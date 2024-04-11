---
title: "客诉AI解决方案设计思路分享" #标题
date: "2024-04-11" #创建时间
author: ["Plutoxx8"] #作者
categories: 
- AI
tags: 
- prompt
description: "" #文章描述
weight: # 输入1可以顶置文章，用来给文章展示排序，不填就默认按时间排序
slug: "" #seo使用，示例：http://example.com/ultimate-guide-making-perfect-pasta
draft: false # 是否为草稿
comments: true #是否展示评论
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: true #顶部显示当前路径
cover:
    image: "" #图片路径：posts/tech/文章1/picture.png
    caption: "" #图片底部描述
    alt: "" #这里填写图片无法显示时的替代文本
    relative: false # 如果这个路径是从网站根目录开始的，这里就保持为 false
---

# 一、前言
AI的快速发展及展现出来的爆炸效果，使得很多人都想把AI能力应用到项目中。目前行业上落地比较好的是Chatbot、工具、虚拟角色等。
3月国内外AI产品榜单（仅列出top10）：

![排行榜单](https://blogpicxx8.oss-cn-shanghai.aliyuncs.com/%E6%8E%92%E8%A1%8C%E6%A6%9C%E5%8D%95.jpg)
详细榜单可查看：[https://dnipkggqxh.feishu.cn/wiki/J0NMwIIPqipbBhkcz7Ycz7jpnBg]

客诉是人与人之间的沟通，基本是“用户”描述诉求，“客服”查询信息给予回复来解决问题，命中了chat的场景。而LLM对于信息查询、一定的推理都比较擅长，所以很多公司对使用LLM来降低客诉成本这件事寄予厚望，认为**80%** 的客服都可以通过LLM来替代。
引用一段内容来描述目前及未来的客诉+LLM演进方向是比较合适的。

>**L1阶段：** 纯坐席人工回复，即为前面所达的传统客服对话工作流。  
>**L2阶段：** 以坐席人工回复为主，LLM回复作为辅，称为“话术推荐”。  
>**L3阶段：** 以LLM回复为主，坐席在旁监督，必要时会“踩刹车”接管。  
>**L4阶段：** LLM完全像人类一样能独立承接好用户的诉求，理想情况下技术能达到的终极目标

L2阶段主要是给“客服”人员提效；  
L3阶段则是转换“客服”工作，变更为监督，因为LLM有幻觉，直接跟用户对话的话，生成内容不可控，对于客诉这种需要明确答案的场景比较致命，所以需要监管；  
L4阶段是随着LLM能力的发展，也许在某些场景幻觉可以消失，或者有控制幻觉的技术手段。
目前我们探索的方案属于L4阶段，尝试用LLM来代替部分人类。

解决思路：
- 短期：内化模型，减少模型与用户的交互，用户看到的只是前端解决方案页面，以此来简化用户操作流程及减少幻觉。
- 中长期：利用agent能力，通过感知周围环境信息去自主决策、执行，选择应该查询的信息或者规则生成内容回复给用户，解决用户问题。

短期方案应用在了外卖柜的客诉问题解决上，下面我会详细介绍这种方案的设计思路。
# 二、框架
![框架](https://blogpicxx8.oss-cn-shanghai.aliyuncs.com/%E6%A1%86%E6%9E%B6.jpg)
- **定位问题：** 通过分析客诉数据来识别问题类型和占比，同时考虑ROI确定优先级。定位问题的方式很多，下面会详细介绍使用ChatGPT获取上述信息的方式。
- **概括场景：** 概括场景的目的是通过已定位的问题，获取相关场景的解决方案、规则和知识库信息。理解各场景的特定需求和判断条件。
- **AI方案设计：** 根据问题的定位和场景的概括，设计AI方案。包括每个场景具体的prompt和输入字段及输入字段规则，确保AI可以精确识别并应对各种客诉。
- **效果校验：** 设计完成后，选取测试集来检验方案的实际表现。根据测试反馈进行调整，优化方案以提高识别和处理的准确性。
# 三、详细思路
## （一）定位问题
通常在定位问题时，需要通过大量的客诉分析和听音来更全面地了解当前的“问题类型现状”和“客服处理结果现状”，从而减少客服概括错误或标注错误带来的影响。使用ChatGPT来分析数以万计的客诉可以显著节省时间；但分析完成后，**还需进行人工复核以确保分析的准确性**。  
首先，我们利用SQL筛选出需要分析的客诉类别，并提取出“反馈问题”及“处理结果”这两个关键数据字段。然后，将这些数据以Excel格式提供给ChatGPT，请求其进行详细分析。

**1、prompt模板**
```
"角色"+"擅长能力"+"背景介绍"+"你的诉求"
##分析要求及步骤
```
注：
- prompt中的“角色”设定是为了提升模型的互动质量和用户体验，让模型以设定的角色视角来回复。    
- “背景介绍”是为了告诉模型现在是什么情况，类似你要求别人做事情时候需要告诉别人背景，别人才能给出更符合你问题的回复。
- “分析要求及步骤”是为了告诉ChatGPT，你需要怎么分析，具体需要“它”执行什么步骤，如果有明确的分析分类，甚至可以给出分类定义，会得到相对更高质量的回复。可以适当增加“示例”来告诉模型你想要的是什么样的分析。
- “##”是分隔符，提高prompt的可读性。
- **prompt不是一步到位的，你需要跟模型有持续的互动，不停调整prompt来达到想要的效果。**

**2、prompt示例**
以下是我在分析外卖柜客诉问题时的prompt，供参考。
```
你是优秀的数据分析专家，善于利用各种手段来通过数据分析问题。
“取餐码错误 3.27”是一个关于外卖柜客诉的Excel，“工单问题”代表了这些客诉工单的类别；“反馈内容”代表了用户来电的事件概要；“处理结果”代表了客服对于这件事的处理结果。
你需要先进行解析，再进行分析、概括。

##你需要分析以下内容
###1、用户基本都在反馈什么事件？你来做总结概括，其中涉及到的数字的部分，需要经过你的一步一步计算，而不是随便给出，概括样式见示例。 
示例：""" 从反馈内容来看，用户基本上都在反馈关于无法开启具体地点的外卖柜、外卖柜离线或格口问题以及超时未取餐导致的锁定情况。 65%以上的客诉事件是在反映“无法开启具体地点的外卖柜”问题，针对这些内容的客服的处理结果通常是“核实格口空闲开柜”。 其中也有少部分的“离线”问题，这部分问题，客服处理结果90%以上是“升级TT”。 还有一部分是因为“格口锁定”问题所以无法开柜，这部分问题客服的处理结果绝大部分是“核实用户的取餐码后，联系运营，确认能否开柜”。 剩下一些是复合的客诉事件，包含了以上几种问题的混合。 """ 
###2、你需要分析客服的解决方案占比。 客服通常处理结果：
1）开柜（核对取餐码开柜、联系运营后开柜、空闲开柜都属于协助用户开柜）；
2）升级客服（客服本人无法解决，需要升级给到二线客服决策，但如果最后包含了）；
3）联系骑手/商家（客服本人要求用户去联系骑手或者联系商家）； 
4）赔付（用于用户丢餐的处理，会有一定的赔偿） 
5）其他处理结果，你来概括。
```

**3、结果展示**
模型对于数据计算不会那么准确，文本过长则不会分析上下文，基本只会按照关键词来进行分析处理，所以**不要完全相信模型给出的数字，只作为参考即可。** 还需要结合听音等其他信息综合分析。  

![输出结果](https://blogpicxx8.oss-cn-shanghai.aliyuncs.com/%E7%BB%93%E6%9E%9C%E5%B1%95%E7%A4%BA.jpg)
## （二）概括场景
通过还原用户场景，了解客服处理方案、SOP、规则等内容综合概括。
客服处理方案部分跟上面定位问题时使用的方式一致，在此就不展开了。
## （三）AI方案设计
设计方案时首先需要明确这个prompt为了解决什么问题（一般根据「定位问题」时可以明确具体问题占比，进一步分析问题场景）。  
以外卖柜客诉来举例，通过大量客诉及听音发现外卖柜客诉中的用户大部分是为了开柜取回自己的物品。
因此，一期的prompt设计旨在解决用户在特定场景下的开柜问题，然后我们需要思考什么场景什么条件可以开柜，并结合系统已有的信息进行校验，不需要用户额外输出过多内容。

**外卖柜中文prompt**
```
你是外卖柜专业客服人员，对于处理是否协助用户打开柜门的问题身经百战，针对每个问题都能一步一步推理给出妥善的解决方案。从不主观臆断，你深刻理解"开柜规则判断标准"，并可以针对不同规则组合快速响应，给出是否开柜或提供人工服务的解决方案。

目的：提供是否协助用户开启柜门的解决方案。
输出格式：不需要思考过程，只需要返回一个JSON对象，包含决策结果的键值对，键为'解决方案'。
---
##开柜规则：
1. 柜机检查
- 在**错误的柜机前**，返回 { "解决方案": "人工服务" }
- 柜机在线状态检测
    - 如果柜机**离线**，返回 { "解决方案": "人工服务" }
    - 如果柜机在线，执行后续流程。
2.是否被运维人员清理
- 是，返回 { "解决方案": "人工服务" }
- 否，继续其他判定流程。
3. 用户信息和订单匹配
- 如果订单**未取**，协助开柜
- 格口状态：
    - **非空闲**，协助开柜
    - **锁定**，返回 { "解决方案": "人工服务" }
    - **已取且空闲**，协助开柜
    - **已取但非空闲**，返回 { "解决方案": "人工服务" }
4. 用户信息与订单不一致或无存件记录
- 格口状态检查：
    - 如果格口**空闲**，可协助开柜。
    - 如果格口**非空闲**或**不知道格口号**，核对取餐码、骑手存餐时间
        - 任意一个一致，协助开柜
        - 全部不一致，返回 { "解决方案": "人工服务" }         
5. 双面柜情况
如果是双面柜，在返回解决方案后，增加 { "柜机类型"：双面柜}
```

**外卖柜英文prompt**
```
You are a professional customer service representative for a food delivery locker service, skilled in determining whether to assist users with opening locker cells. Your approach to each issue is systematic, and you avoid making subjective judgments. You possess an in-depth understanding of the "Locker Cell Opening Decision Criteria" and are adept at quickly responding to various combinations of rules, making decisions about opening the locker cells or providing manual assistance.

## Purpose: To decide whether to assist users in opening locker cells.  
## Output Format:No need for the thought process, only the results in the form of key-value pairs are required, with the key being 'Solution'.  
- Cabinet opening result returned: { "Solution": "Assist in Opening Locker" }  
- Manual result returned: { "Solution": "Manual Service" }  
## Locker Cell Opening Rules:  
1. Locker Machine Check  
If at the wrong locker machine, return "Manual result"  
Check the online status of the locker machine  
If the locker is offline, return "Manual result"  
If the locker is online, proceed to the next steps.  
2. Cleaned by Maintenance Staff  
If yes, return "Manual result"  
If no, continue with additional assessments.  
3. Matching User Information and Order  
If the order has not been collected, assist in opening the locker cell  
Status of the locker cell:  
If not idle, assist in opening  
If locked, return "Manual result"  
If collected and idle, assist in opening  
If collected but not idle, return "Manual result"  
4. Inconsistency in User Information and Order or No Deposit Record  
Check the status of the locker cell:  
If the cell is idle, assist in opening  
If the cell is not idle or the cell number is unknown, verify the meal pickup code, the rider's meal deposit time, or the rider's mobile number  
If any of these match, assist in opening  
If none match, return "Manual result"  
5. Double-sided Locker Situation  
In the case of a double-sided locker, add { "Locker Type": "Double-sided" } after providing the solution.
```
这个prompt设计有几个核心要点：
- 写完中文prompt时，需要再写个英文的prompt进行最终调用，有两个原因：第一，对于ChatGPT来说英文prompt回复质量更高；第二英文prompt的token更少，费用更低。外卖柜这个prompt的版本来回一次调用仅花费0.1元。
- 需要定义对应的输入字段，结合输入字段传入最终判断结果。
- 在进行prompt设计时，需要明确大模型输出格式。因为我们的目标是输出解决方案并指引用户到对应的前端解决页面，所以不需要生成大段的回复内容。
- 为了提高prompt的回复质量可以使用一些prompt技巧，如CoT、Reflection等，下面“扩展”的部分会讲到。
## （四）效果校验
prompt设计完成后需要进行校验，判断一定量级的真实场景下，模型是否还能做出准确的判断。如果不可以，需要再对prompt进行定向调优。  

**1、准备工作**  
1）选定测试集：从prompt要解决的各个开柜场景都选取一些客诉内容。  
2）场景构建：输入的条件其实就是场景，所以我采用的方式是又写了一个prompt，根据外卖柜客诉的ASR内容生成对应字段，还原真实场景。  
3）测试集人工标注：需要找客服/你自己针对你要测试的客诉内容进行标注，比如外卖柜这条客诉的原本处理结果是“开柜”还是“联系骑手”等。  

**2、校验**  
调用gpt接口，使用开柜prompt和“准备工作”生成的场景进行客诉处理结果生成。  
再使用生成结果与“准备工作”的人工标注进行结果比对，来判断准确率，结合不准确的场景再进行定向调整。
# 四、总结
看完以后你肯定认为if/else的逻辑也能写出来啊，为什么非要用AI？  

首先，解决问题的手段很多，AI只是其中之一，但如果AI能快速验证场景及收益，那先用AI验证未尝不可。  
其次，AI的扩展性更好，如果要新增场景只需要对prompt做相关更改即可覆盖。  
最后，这种判断方式一定要跟前端做结合，我们通常遇到的客诉基本都是出来一个list让你选，外卖柜也不例外，做这件事的初衷也是希望可以直接用AI定位用户的问题引导到指定的解决方案，减少用户的选择，真实的提高用户体验。
# 五、最后
以及可以从我之前写的文章里看看相关的prompt写作技巧[https://plutoxx8.github.io/blog/posts/prompt%E5%86%99%E4%BD%9C%E6%8A%80%E5%B7%A7/]  

祝大家做强大力量的掌控者🫴
![gogogo](https://blogpicxx8.oss-cn-shanghai.aliyuncs.com/IMG_5282.jpg)