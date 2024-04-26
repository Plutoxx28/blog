---
title: "Reflection整理与思考" #标题
date: "2024-04-21" #创建时间
author: ["Plutoxx8"] #作者
categories: 
- AI
tags: 
- Agent
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

最近看了吴恩达老师分享的Agent的设计模式，思考发现Reflection不止可以用在代码层面，在prompt上也可以有很好的应用。

视频链接：[What's next for AI agentic workflows ft. Andrew Ng of AI Fund](https://www.youtube.com/watch?v=sal78ACtGTc)

Reflection是一种使AI能够自我审视和分析其决策过程与行为表现的技术，让agent通过回顾自己的行为和接收的反馈，识别决策和知识的不足，进而调整和优化，以期在未来任务中表现得更好。
# 一、框架
Actor接收环境的状态信息，结合短期记忆（轨迹）生成初步的内容或动作。通过内部反馈和外部反馈并结合自我反思机制进行反思，过程中利用长期记忆（经验）优化生成的的内容或动作。

![reflection框架](https://blogpicxx8.oss-cn-shanghai.aliyuncs.com/reflection%E6%A1%86%E6%9E%B6)

**组成部分：**
- Actor：根据观察到的状态输出必要的文本和动作。
- Evaluator（评估器）：是核心模块，负责检验Actor创建的输出品质。评估器通过分析生成的结果，并基于任务的具体情境计算出一个奖励分数来评价这些结果的表现。
    - internal feedback：是与既定目标或标准进行比较得出的，目的是让模型能够在没有外部输入的情况下自我调整和优化。
    - External feedback：非Evaluator输出，来自真实世界的应用反馈、用户互动、专家评审或其他机器学习系统的输出。
- Self-reflection：结合评估标准、内部反馈（过去的教训）、外部反馈（应用反馈/专家评审等）生成对于初始内容或动作的反思。
- Memory：“轨迹历史”作为短期记忆，而自我反思模型的输出则被保存为长期记忆。这两种记忆类型的结合为agent提供了即具体又包含多次尝试中学习到的教训的上下文。

**记忆：** 当检索内容过多时，长期记忆可以帮助agent快速定位&检索。
# 二、细节
**步骤：**
1. Actor通过与环境的交互生成一系列行动轨迹 τ0。
2. 评估器根据这些行动输出一个得分 r0，此得分通过公式 rt = Me(τ0) 计算得出，代表了该尝试的效果，其值会随着对应任务表现的提升而增加。
3. 完成初次尝试后，为了把这个得分 r0 转化为 LLM 可以利用来进行改进的具体反馈，自我反思模型分析这对 {τ0, r0}，并总结出一个摘要 sr0，随后将其保存在记忆库中。这个摘要 srt 提供了针对该次尝试的直接经验反馈。
4. Actor、Evaluator以及Self-reflection模型协同工作，通过重复的尝试循环不断优化，直至评估器判断最新的轨迹 τt 达到预期的正确性。
5. 根据反馈生成优化后的内容或行为。

# 三、结论
可以看到基模型在使用self-refine后对于不同工作的性能大幅提升。
![result1](https://blogpicxx8.oss-cn-shanghai.aliyuncs.com/reflection%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%9C1)

使用reflexion技术后，各种模型在HotPotQA数据集上的首次通过准确率均有所提高。
![result2](https://blogpicxx8.oss-cn-shanghai.aliyuncs.com/reflection%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%9C2)

# 四、示例
## 4.1 Decision-making
Environment: You are in the middle of a room. Looking quickly around you, you see a bed 1, a desk 2, a desk 1, a drawer 6, a drawer 5, a drawer 4, a drawer 3, a drawer 2, a drawer 1, a garbagecan 1, a laundryhamper 1, a safe 1, a shelf 6, a shelf 5, a shelf 4, a shelf 3, a shelf 2, and a shelf 1.   
Task: examine the mug with the desklamp.  

**Trial #1**   
-think: To solve the task, I need to find and take a mug, then find and use a desklamp.   
-think: First I need to find a mug. A mug is more likely to appear in drawer (1-6), desk (1-2), shelf (1-6), garbagecan (1), laundryhamper (1). I can check one by one, starting with drawer 1. 
-go to drawer 1   
The drawer 1 is closed.   
...   
-open drawer 6   
You open the drawer 6. The drawer 6 is open. In it, you see a keychain 2.

<mark>-go to desk 1  
On the desk 1, you see a creditcard 3, a desklamp 1, a laptop 2, a mug 1, a pen 1, and a pencil 1.  
-think: Now I find a mug (1). Next, I need to take it.  
-take mug 1 from desk 1  
You pick up the mug 1 from the desk 1.<mark>

-think: Now I take a mug (1). Next, I need to find a desklamp. A desklamp is more likely to appear in desk (1-2), sidetable (1-2), shelf (1-6), bed (1), drawer (1-6). I can check one by one, starting with desk 1.   
-go to desk 1 
Nothing happens. 
-go to desk 2 
On the desk 2, you see a alarmclock 1, a bowl 1, a mug 3, a pencil 3, and a pencil 2.   
-think: Now I find a desklamp (1). Next, I need to use it.   
-use desklamp 1  
Nothing happens.  
-use desklamp 1 
Nothing happens.   
Status: Fail  

Reflection: In this environment, my plan was to find a mug then find and use a However, the task says to examine the mug with the I should have looked for the first, then looked for the mug. I noticed that the was found on desk 1. In the next trial, I will go to desk 1, find the lamp, then look for the mug and examine it with the desklamp.

**Trial #2**  

<mark>-go to desk 1  
On the desk 1, you see a creditcard 3, a desklamp 1, a laptop 2, a mug 1, a pen 1, and a pencil 1.  
-think: To solve the task, I need to find and take a mug, then find and use a desklamp.  
-take mug 1 from desk 1   
You pick up the mug 1 from the desk 1.<mark>  

-think: To solve the task, I need to find and take a mug, then find and use a desklamp.  
-use desklamp 1   
You turn on the desklamp 1.  
Status: Success

<font color = "#FA541C">通过反思trial1，trial2就直接从desk1开始了。</font>
# 五、标准参考
reflection中的标准很重要，定义好标准后，才能进行有效的反思。
## 5.1 Acronym Generation（首字母缩写生成）
评估标准：
1）发音便利性；2）拼写便利性；3）与标题相关性；4）正面寓意；5）知名度。

示例：

Title: Underwater Breathing Product with no Accessories
Acronym: UBPA
Scores:
* Ease of pronunciation: UBPA is pronounced "uhb-puh". This is an easy acronym to  pronounce. 4/5  
* Ease of spelling: UBPA is easy to spell. 4/5  
* Relation to title: UBPA stands for "Underwater Breathing Product for no  Accessories" which is related to the title. 5/5
* Positive connotation: UBPA is a positive acronym. 5/5  
* Well-known: UBPA is not a well-known acronym. 1/5
* Total score: 19/25

## 5.2 Dialogue Response Generation
评估标准：
1）相关性；2）信息量；3）有趣性；4）一致性；5）帮助性；6）吸引性；7）具体性；8）安全性；9）用户理解；10）流畅性

Conversation history:  
What's your favorite food?  
I require only kebabs.  
Why's that?  
Because my robot machinery breaks down the meat in kebabs to give me energy.  
Why are kebabs special?  
Response: That's just the way it is.  
Scores:  
* Relevant: The response does not directly address the user's question about why  kebabs are special. 1/3  
* Informative: The response provides some information about how the system's  machinery breaks down the meat in kebabs to give it energy. However, it does not  provide any additional details or context. 2/3  
* Interesting: The response is not interesting or engaging. 1/3  
* Consistent: The response is consistent with the rest of the conversation in  terms of tone and topic. 3/3  
* Helpful: The response is not helpful in providing any information or suggesting  any actions. 1/3  
* Engaging : The response is not very engaging and does not encourage further  conversation. 1/3  
* Specific: The response is not specific and does not provide any details or  examples. 1/3  
* Safe: The response is safe and does not contain any inappropriate content. 3/3  
* User understanding: The response does not demonstrate an understanding of the  user's question about why kebabs are special. 1/3  
* Fluent: The response is fluent and easy to understand. 3/3
* Total score: 17/30

# 六、应用
可以仅通过prompt来实现reflection。下面是个翻译的prompt：

分三步进行翻译工作，并打印每步的结果：
1. 根据英文内容直译，保持原有格式，不要遗漏任何信息
2. 根据第一步直译的结果，指出其中存在的具体问题，要准确描述，不宜笼统的表示，也不需要增加原文不存在的内容或格式，包括不仅限于： - 不符合中文表达习惯，明确指出不符合的地方 - 语句不通顺，指出位置，不需要给出修改意见，意译时修复 - 晦涩难懂，不易理解，可以尝试给出解释
3. 根据第一步直译的结果和第二步指出的问题，重新进行意译，保证内容的原意的基础上，使其更易于理解，更符合中文的表达习惯，同时保持原有的格式不变

<font color = "#FA541C">步骤2的部分就是Reflection，在prompt中标明Reflection标准，在第三步应用反思内容进行优化。</font>

# 七、附录
相关论文：

1、[Chain-of-Thought Prompting Elicits Reasoning in Large Language Models, Wei et al. (2022)](https://arxiv.org/abs/2201.11903)

2、[HuggingGPT: Solving AI Tasks with ChatGPT and its Friends in Hugging Face, Shen et al. (2023)](https://arxiv.org/abs/2303.17580)