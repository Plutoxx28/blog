---
title: "研究了 OpenClaw 的记忆系统，发现它其实就做了一件事" #标题
date: "2026-02-12" #创建时间
lastmod: "2026-02-12"
author: ["Plutoxx28"] #作者
categories: 
- AI Agent
tags: 
- OpenClaw
- Memory
description: "拆解 OpenClaw 记忆系统：Markdown 持久化 + BM25/向量混合检索 + flush-before-compact，一套让 Agent '记得住'的完整流水线。" #文章描述
weight: # 输入1可以顶置文章，用来给文章展示排序，不填就默认按时间排序
slug: "openclaw-memory-system-explained" #seo使用，示例：http://example.com/ultimate-guide-making-perfect-pasta
draft: false # 是否为草稿
comments: true #是否展示评论
ShowToc: true # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
ShowBreadCrumbs: true #顶部显示当前路径
ShowReadingTime: true #展示阅读时间
cover:
    image: "" #图片路径：posts/tech/文章1/picture.png
    caption: "" #图片底部描述
    alt: "" #这里填写图片无法显示时的替代文本
    relative: false # 如果这个路径是从网站根目录开始的，这里就保持为 false
---

你可能遇到过这种情况：上周跟 AI 助手聊了半天项目方案，这周再问“当时怎么定的？”，它一脸茫然。这不是它笨。LLM 的“记忆”本质上是上下文窗口（Context）——模型在一次请求里能看到的全部内容，由系统提示、对话历史、工具输出和你刚发的消息拼起来。模型的推理和回答只能基于 Context 里有的东西，但 Context 是短暂的、有上限的、越大越贵。你希望它像硬盘一样长期存储，但它的工作方式决定了它做不到。

最近在研究 OpenClaw，发现它的记忆系统解决这个问题的思路特别清晰，核心就一句话：

**别指望模型在上下文里永远记得。把重要的东西写到文件里，需要的时候用检索找回来。**

听起来朴素，但它把“写什么、怎么写、怎么找、快撑爆了怎么办”这一整套流程设计得很完整。先从“记忆长什么样”开始：它不是某个黑盒功能，而是一套能落盘、能检索、能在长对话里自我维护的工作流。

## 一、记忆长什么样：两本笔记 + 一组身份文件

### 1、记忆就是 Markdown 文件

OpenClaw 里的 Memory（记忆）是落在磁盘上的持久化内容，跟 Context 是两回事——Context 随对话结束就没了，Memory 不会。但模型不会自动“看到”Memory 里的内容，需要通过检索工具把相关片段拉回到 Context 里才能用。

具体形态上，OpenClaw 的记忆不是神秘数据库，就是工作区里的 Markdown 文件。官方文档原话：
> **files are the source of truth; the model only “remembers” what gets written to disk.**

它分两层，像两本笔记本：

- **`memory/YYYY-MM-DD.md`** —— 流水账。按天记录，追加写入，适合“今天发生了什么”“这周在推什么”。默认模板会要求 agent 在 session start 主动阅读今天和昨天的日记；同时索引器也会扫描 `memory/` 目录，用于后续检索建库。

- **`MEMORY.md`（可选，不会自动创建，需要你主动建或让 agent 写入）** —— 精华本。放长期偏好、关键决策、稳定不变的事实。但有个重要细节：它只在私聊 session 中注入，群聊/频道里不会加载，避免私人记忆泄露到公共对话。

### 2、每次对话都注入的“身份文件”

记忆文件解决的是“跨 session 记住发生过什么”，但 Agent 还需要知道“我是谁、用户是谁、该怎么做事”。OpenClaw 在每次 session 开始时会自动注入一组工作区文件到系统提示里，它们和记忆文件一起，构成了 Context 里跨 session 持续存在的部分：

- `AGENTS.md` —— 行为规则、协作约定
- `SOUL.md` —— 人格、语气、风格
- `TOOLS.md` —— 工具使用说明
- `IDENTITY.md` —— 名字、角色
- `USER.md` —— 用户画像、偏好
- `HEARTBEAT.md` —— 定时任务（heartbeat）指令
- `BOOTSTRAP.md` —— 首次运行的初始化脚本（只在全新工作区创建；按模板约定，完成后应删除，避免后续重复注入）

空文件会跳过，过大的文件会被截断。截断策略是保留 70% 头部 + 20% 尾部 + 中间插截断标记。这么做是因为文件开头通常是规则定义（最重要），末尾通常是最新追加的内容（次重要），中间可以牺牲。

还有个设计值得注意：当 OpenClaw 创建子 Agent 处理子任务时，只注入 `AGENTS.md` 和 `TOOLS.md`，其他文件全部跳过。子任务不需要完整人设，精简注入省 token。到这里我们解决了两个问题：“记忆放在哪”、“每次请求会带哪些长期设定”。下一步就该问：这些东西是怎么被写进去的？

## 二、“记住”这件事怎么发生：写入从哪来

OpenClaw 不会凭空产生记忆，它的“长期记住”来自写文件。写入来源通常有三种：

- **你明确说“记住这个”**。助手就写进 `MEMORY.md` 或当天的 `memory/YYYY-MM-DD.md`。官方建议是：如果你想让某件事被记住，直接让 bot 写到 memory 里。

- **助手自己判断“这值得记”**。比如你在对话中确认了一个关键配置、一个设计决策，助手可能主动落盘。

- **系统在关键时刻提醒它“现在该存档了”**。这是后面要讲的 Memory Flush。

这里的关键不是“OpenClaw 记忆很强”，而是它把行为设计成了：能写就写，写完可检索。这比“在对话里口头说过一次”靠谱得多。写进文件只是第一步。下一步才是关键：当你换一种问法，它还能不能把那段内容找回来？

## 三、写完怎么找回来：索引 + 混合检索

有一个前提需要先说清楚：OpenClaw 的检索不是自动发生的。`memory_search` 是一个工具（tool），模型需要根据对话上下文主动判断“这个话题我可能记过”，然后决定调用它。这跟传统 RAG 的“每次请求都自动塞入相关片段”不同——OpenClaw 不会在每次对话中都注入无关的记忆内容，该查的时候才查，省 token。

理解了这一点，再看索引和检索的设计就更清楚了。如果只有一堆 Markdown 文件，没有索引，你会遇到两个问题：文件越来越多，不知道在哪天哪段；你的问法一变，它就找不到——你问“那个数据库连接串”，文件里写的是 `POSTGRES_URL`。

所以 OpenClaw 在写文件之外，还做了索引：监听文件变更 → 切片 → 做 embedding → 建本地 SQLite 索引 → 检索时混合打分。

### 1、文件监听：不是改了就重建

OpenClaw 用 chokidar 盯着 `MEMORY.md`、`memory/` 和配置的额外路径。变更后有 1.5 秒的 debounce（等文件写稳定），同步时用 SHA-256 对文件做 hash，内容没变就跳过，不重新切片。如果 embedding 模型或切片参数变了，会触发全量重建索引。重建时用临时数据库做完所有工作再原子交换过来，中途失败不影响原库。

### 2、切片：400 tokens 一段，80 tokens 重叠

Markdown 按行累积切片，每段大约 400 tokens，相邻段有 80 tokens 重叠。重叠是为了防止一条“决策”跨段落边界，检索只拿到一半。

### 3、混合检索：向量 + BM25 关键词

这是我觉得设计最实用的部分。OpenClaw 同时用两种检索：

- **向量检索**擅长找“意思相近”的内容。你问“数据库连接”，它能匹配到 `POSTGRES_URL`、`connection string`、`DB url` 这些不同说法。

- **BM25 关键词检索**（一种经典的全文检索算法，按关键词的出现频率和稀有程度打分）擅长找“字面命中”的内容。人名、日期、ID、配置键、代码符号这些，向量反而不如关键词靠谱。

合并方式是：两边各取一个候选池（默认是最终结果数的 4 倍），按 chunk ID 做并集合并，然后加权打分。默认权重是 `0.7 × 向量 + 0.3 × 关键词`，低于 0.35 分的结果过滤掉，最终返回前 6 个。官方文档自己也说了，这不是“IR 理论上最完美的方案”，但简单、快、在实际笔记场景下效果不错。

补充一点：默认情况下只有 memory 文件会被索引，但 OpenClaw 还有一个实验性功能——**Session 转录索引**。要用它，开启后（`experimental.sessionMemory = true`，还需要把 `sources` 里包含 `"sessions"`，如 `sources: ["memory", "sessions"]`），这样即使你没让助手“记住”某件事，只要对话中讨论过，`memory_search` 也有可能通过语义检索找到那段对话。相当于把检索范围从“主动写下的笔记”扩大到了“所有聊过的内容”。

到这里，“写”和“找”都打通了。但还有一个现实挑战：对话越长，上下文越容易被撑爆。OpenClaw 怎么处理这个？

## 四、长对话怎么不炸：三个机制

把 Agent 当长期伙伴用，对话一定会变长。OpenClaw 用三层策略来治理上下文膨胀。

### 1、Compaction：把早期历史压成摘要

当 session 的上下文快到模型窗口上限时，OpenClaw 触发 compaction：把较早的历史总结成一条摘要，保留最近的消息。

好处直接：token 用量拉下来，会话能继续。但副作用也直接：摘要是有损的，细节会被压缩掉。比如一段很长的对话经过 Compaction 后可能变成：

```
[Compaction 摘要]
用户和助手讨论了数据库迁移方案，从 PG-A 迁到 PG-B，确认了连接串。
讨论了是否使用 GraphQL，最终决定不用。还调试了一个 CORS 配置问题并解决。
```

信息是对的，但具体的连接串是什么、不用 GraphQL 的理由是什么、CORS 怎么配的——这些细节全丢了。

### 2、Memory Flush：压缩之前先存档

这是我觉得 OpenClaw 最聪明的设计。

它不指望 compaction 做到无损，而是在 compaction 真正发生之前，插入一个静默的 agentic turn，让模型自己判断"哪些具体事实值得留下来"，然后以原始粒度写入文件。注意，Flush 不是在做摘要，而是在做"挑拣+落盘"——模型会把具体的值、具体的理由、具体的结论写下来。

同样的对话，Flush 写入 `memory/2026-02-11.md` 的内容可能是：

```
## 数据库迁移
- 从 PG-A 迁移到 PG-B，新连接串：POSTGRES_URL=postgresql://admin:xxx@pg-b.internal:5432/prod
- 决定不用 GraphQL，理由：现有 REST API 已覆盖所有场景，团队没有 GraphQL 经验，引入会增加维护成本
- CORS 问题：nginx 配置里 Access-Control-Allow-Origin 要用具体域名，不能用通配符 *（生产环境安全要求）
```

区别一目了然：**Compaction 压缩的是对话历史，丢细节换空间；Flush 保存的是具体事实，后续可以被精确检索回来。** 具体来说，系统会同时注入两条提示：一条追加到 system prompt，告诉模型当前 session 即将压缩，应该把值得长期保存的内容写到磁盘；一条作为 user prompt，指示模型将笔记写入 `memory/YYYY-MM-DD.md`，如果没什么需要保存的就回复 `NO_REPLY`。

触发时机是 `contextWindow - reserveTokensFloor(20000) - softThresholdTokens(4000)`。比如 200K 上下文的模型，大约在 176K tokens 时就会触发 flush，比真正的 compaction 提前了 4000 token 的量。每个 compaction 周期只 flush 一次，防止重复。用户完全感知不到这个过程。

**为什么这步很关键？** 因为它把风险从"摘要是否写对"转成了"有没有在压缩前把关键点落盘"。一旦落盘，哪怕摘要漏了，后续通过 `memory_search` 还能检索回来。**一句话：先存档，再压缩。**

### 3、Session Pruning：专治工具输出太肥

Pruning 解决的是另一类膨胀：工具输出。Agent 在工作过程中会调用各种工具（执行命令、读文件、搜索等），这些工具的返回结果（toolResult）会原样留在上下文里。一个 `exec` 跑出几万行日志、一个 `read` 拉回一整个配置文件，堆积起来会快速耗尽 token。

Pruning 的策略很克制：只修剪 toolResult，不动用户和助手的正常消息；而且只影响这次发给模型的上下文，不回写磁盘上的对话记录。分两级：**Soft-trim**——保留头部和尾部，中间用省略号替代。适用于内容虽大但可能有用的输出：

```
# 修剪前（原始 toolResult，假设 3000 行日志）
[2026-02-11 10:01:03] Starting migration...
[2026-02-11 10:01:03] Connecting to source DB...
[2026-02-11 10:01:04] Reading table users (150,000 rows)...
[2026-02-11 10:01:05] Reading table orders (2,300,000 rows)...
...(中间 2990 行省略)...
[2026-02-11 10:15:22] Migration completed. 45 tables, 0 errors.

# Soft-trim 后
[2026-02-11 10:01:03] Starting migration...
[2026-02-11 10:01:03] Connecting to source DB...
...
[2026-02-11 10:15:22] Migration completed. 45 tables, 0 errors.
```

**Hard-clear**——整个替换成一句提示。适用于更老的、大概率不再需要的输出：

```
# Hard-clear 后
[Old tool result content cleared]
```

可以理解成：不删档案，只是这次开会别把 5 万行日志打印出来。

现在把这些机制串起来，才是完整的"记忆系统"长什么样。

## 五、一次完整流程长什么样

先看整体数据流：

```mermaid
graph LR
    A[用户对话] -->|"记住这个"| B[写入 memory/*.md]
    B -->|chokidar 监听| C[切片 + Embedding]
    C --> D[(SQLite 索引)]
    
    E[用户提问] -->|Agent 主动调用| F[memory_search]
    F --> D
    D -->|混合检索结果| G[注入 Context → 回答]
    
    H[Context 快满] -->|提前 4000 token| I[静默 Flush → 写入 memory]
    I --> B
    H -->|到达上限| J[Compaction 压缩摘要]
```

<br>
把上面的机制串起来，一个典型场景是这样的：

<div style="border:1px solid rgba(128,128,128,0.3); border-radius:8px; padding:20px; margin:16px 0;">

**你**: 把数据库从 A 迁到 B，连接串是 `POSTGRES_URL=xxx`，记一下。

**助手**: 好，写入 `memory/2026-02-11.md`。

> → 文件变更 → 1.5s 稳定 → hash 比对 → 重新切片 + embedding → 更新索引

*...聊了很多，跑了很多命令...*

**（接近 flush 阈值）**

**系统** \[静默\]: Pre-compaction memory flush...

**助手** \[静默\]: `NO_REPLY`（或写入遗漏的关键信息）

**（随后 compaction）**

> \[摘要\] 迁移了数据库，确认了连接串，决定不用 GraphQL 的理由是...

**你**: 当时为什么不用 GraphQL？

**助手**:
> → `memory_search("GraphQL 决策")`
> → 向量: 0.82 分命中 `memory/2026-02-11.md`
> → BM25: 0.91 分命中同一 chunk（“GraphQL”精确匹配）
> → 混合分: `0.7×0.82 + 0.3×0.91 = 0.847`
> → 召回 + 回答

</div>

整个“记得住”不是玄学，是一套流水线：写入 → 索引 → 检索 → 在压缩前补写。

## 六、回到开头那句话

OpenClaw 的记忆系统确实就做了一件事—— **把模型的“短期记忆”卸载到文件系统，然后用检索桥接回来。** 它没有用什么黑科技，Markdown 文件、SQLite 索引、BM25 + 向量混合检索，都是成熟的技术。但它把这些东西组合成了一条完整的流水线：写入 → 监听 → 索引 → 检索 → 在压缩前补写，每个环节都有对应的容错机制。

如果你在做 AI Agent 相关的产品，这套思路值得参考——不是因为实现复杂，恰恰是因为它足够简单和透明。记忆就是文件，出了问题打开就能看、能改、能用 git 追溯。这可能是它最聪明的设计选择。

## 附：关键配置一览

如果你想调参，这些是默认值：

| 参数 | 默认值 | 含义 |
|------|--------|------|
| chunk tokens | 400 | 每个切片的目标 token 数 |
| chunk overlap | 80 | 相邻切片的重叠 token 数 |
| maxResults | 6 | memory_search 最多返回几个结果 |
| minScore | 0.35 | 低于此分数的结果被过滤 |
| vectorWeight | 0.7 | 向量检索在混合分中的权重 |
| textWeight | 0.3 | BM25 在混合分中的权重 |
| candidateMultiplier | 4 | 候选池是最终结果数的几倍 |
| watchDebounceMs | 1500 | 文件变更的 debounce 时间 |
| reserveTokensFloor | 20000 | compaction 预留 token 数 |
| softThresholdTokens | 4000 | flush 比 compaction 提前触发的 token 量 |

<br>

> 参考：[OpenClaw 官方 Memory 文档](https://docs.openclaw.ai/concepts/memory) ｜ 源码基于 [OpenClaw github仓库](https://github.com/openclaw/openclaw)
> 
> 注：上文所述的索引和检索均基于 OpenClaw 的默认方案——内建 SQLite（配合 sqlite-vec 加速向量查询），零外部依赖、本地优先。此外 OpenClaw 还支持用 QMD（一个本地搜索引擎，在 BM25 + 向量之上增加了 reranking）作为实验性替代后端，配置 `memory.backend = "qmd"` 启用。QMD 挂了会自动回退到内建 SQLite。
