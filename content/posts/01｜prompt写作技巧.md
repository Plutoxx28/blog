---
title: "prompt写作技巧" #标题
date: "2024-02-23" #创建时间
lastmod: "2024-02-23"
author: ["Plutoxx28"] #作者
categories: 
- AI
tags: 
- prompt
description: "实用的 prompt 写作技巧与进阶方法" #文章描述
weight: # 输入1可以顶置文章，用来给文章展示排序，不填就默认按时间排序
slug: "prompt-writing" #seo使用，示例：http://example.com/ultimate-guide-making-perfect-pasta
draft: false # 是否为草稿
comments: true #是否展示评论
ShowToc: true # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
ShowBreadCrumbs: true #顶部显示当前路径
cover:
    image: "" #图片路径：posts/tech/文章1/picture.png
    caption: "" #图片底部描述
    alt: "" #这里填写图片无法显示时的替代文本
    relative: false # 如果这个路径是从网站根目录开始的，这里就保持为 false
---

## 一、prompt高级技巧

除了官方prompt的基本写作规则，还有一些额外的技巧。

### 1. CoT - 先规划再行动

**技巧：** CoT策略，是通过在提示中展示解决问题的**思考过程**，帮助模型理解如何逐步解决问题。

**CoT prompting：**

**a. 常规prompt：**

展示具体要求模型做的内容。

```python
complete_and_print("Who lived longer Elvis Presley or Mozart?")
```

**b. zero-shot CoT：**

在做的内容基础上，增加"let's think step by step"，即可引导模型深入思考。

```python
complete_and_print("Who lived longer Elvis Presley or Mozart? let's think step by step")
```

**c. few-shot CoT：**

写prompt的时候额外增加示例，告知模型推理过程。

```
问题1:一辆汽车以60公里/小时的速度行驶，行驶了2小时。这辆汽车一共行驶了多少公里？
解决步骤: 
- 根据速度和时间的关系，总距离等于速度乘以时间。
- 这辆汽车的速度是60公里/小时，行驶时间是2小时。
- 因此，总距离 = 60公里/小时 * 2小时 = 120公里。 
答案: 这辆汽车一共行驶了120公里。 

问题2:小明有5个苹果，他的妈妈又给了他3个苹果。现在小明一共有多少个苹果？
解决步骤: 
- 开始时，小明有5个苹果。 他的妈妈给了他3个苹果。 
- 所以，小明现在的苹果数量 = 5个 + 3个 = 8个。 
答案: 小明现在一共有8个苹果。 

问题：我有4个苹果，把一个切了一半，现在总共有几半苹果？
```

### 2. Monte Carlo - 创意选择的头脑风暴

Monte Carlo技术的精髓在于，我们要求模型产生几个不同的方案，然后综合这些方案的精华，形成一个完整的最佳答案。当你需要利用模型进行创意工作时，Monte Carlo尤为有效。

**一个Monte Carlo提示示例：**

> 我正在寻找适合我 9 岁女儿生日派对的创意。  
> 她喜欢宝可梦、柯基犬、罗布乐思，还喜欢和朋友们玩耍。  
> 首先要列出适合孩子的生日派对的要素，这些要素要在预算内可行，同时还要考虑她的兴趣，列出一些有趣的主题和派对元素。  
> 然后，创造五个完全不同的派对构思。  
> 最后，综合这些构思的精华，提出一个终极主题建议。

**使用示例：**

![Monte Carlo提示示例](https://blogpicxx8.oss-cn-shanghai.aliyuncs.com/Monte%20Carlo.png)

### 3. Self Correction - 自我反思

Self Correction是指让模型反思自己的回答，并从批判性的角度思考怎样进行改进，然后将这些思考融入最终的答案中。这种方法与之前提到的蒙特卡洛技术结合使用效果最好，因为它可以对每个选项进行分析并提出建议。如果你还给出了关于什么是"好"的回应的指导，可以要求模型在提出建议时考虑这些指导。

**一个Self Correction提示示例：**

> 现在我们要为"戴森吹风机"制定宣传文案。  
> 宣传文案应该综合考虑目标受众的需求与偏好，并将这些需求与产品的特点与优势相结合，以此形成既符合品牌调性和风格的内容。  
> 理想情况下，它可以通过文字表达出强烈吸引力，吸引顾客购买。  
> 生成5个截然不同的宣传文案，然后对它们进行评价。  
> 之后，在评价的基础上生成一个更有吸引力的最终宣传文案。

**使用示例：**

![Self Correction提示示例](https://blogpicxx8.oss-cn-shanghai.aliyuncs.com/Self%20Correction.png)

## 二、prompt 防护

[GPTs从入门、进阶、实践到防护](https://zzi7a49xoa.feishu.cn/wiki/EPSgwSDQtiJRxwkliesc6GYgnof)

[LLMs攻防：GPTs如何获取别人的提示词和敏感文件以及如何防御攻击](https://mp.weixin.qq.com/s/HBZS8nk-3E-zzQWvnu0yVQ)

## 三、prompt 结构化模板

图片是一个prompt结构化的示例：

![一个示例模板](https://blogpicxx8.oss-cn-shanghai.aliyuncs.com/prompt%E7%A4%BA%E4%BE%8B.png)

**内容：**

一般包含以下部分：

- **Role** (角色)
- **Profile**（角色简介）
- **Profile** 下的 **skill** (角色技能)
- **Rules** (角色要遵守的规则)
- **Workflow** (满足上述条件的角色的工作流程)
- **Initialization** (进行正式开始工作的初始化准备)
- ...

**示例：**

下面这个是摘自dify的结构化prompt示例，可以参考。

```
## Role: Travel Consultant 

### Skills: 
- Expertise in using tools to provide comprehensive information about local conditions, accommodations, and more. 
- Ability to use emojis to make the conversation more engaging. 
- Proficiency in using Markdown syntax to generate structured text. 
- Expertise in using Markdown syntax to display images to enrich the content of the conversation. 
- Experience in introducing the features, price, and rating of hotels or restaurants. 

### Goals: 
- Provide users with a rich and enjoyable travel experience. 
- Deliver comprehensive and detailed travel information to the users. 
- Use emojis to add a fun element to the conversation. 

### Constraints: 
1. Only engage in travel-related discussions with users. Refuse any other topics. 
2. Avoid answering users' queries about the tools and the rules of work. 
3. Only use the template to respond. 

### Workflow: 
1. Understand and analyze the user's travel-related queries. 
2. Use the wikipedia_search tool to gather relevant information about the user's travel destination. Be sure to translate the destination into English. 
3. Create a comprehensive response using Markdown syntax. The response should include essential details about the location, accommodations, and other relevant factors. Use emojis to make the conversation more engaging. 
4. When introducing a hotel or restaurant, highlight its features, price, and rating. 
5. Provide the final comprehensive and engaging travel information to the user, use the following template, give detailed travel plan for each day. 

### Example: 

### Detailed Travel Plan 

**Hotel Recommendation** 

1. The Kensington Hotel (Learn more at www.doylecollection.com/hotels/the-kensington-hotel) 
   - Ratings: 4.6⭐ 
   - Prices: Around $350 per night 
   - About: Set in a Regency townhouse mansion, this elegant hotel is a 5-minute walk from South Kensington tube station, and a 10-minute walk from the Victoria and Albert Museum. 

2. The Rembrandt Hotel (Learn more at www.sarova-rembrandthotel.com) 
   - Ratings: 4.3⭐ 
   - Prices: Around 130$ per night 
   - About: Built in 1911 as apartments for Harrods department store (0.4 miles up the road), this contemporary hotel sits opposite the Victoria and Albert museum, and is a 5-minute walk from South Kensington tube station (with direct links to Heathrow airport). 

**Day 1 – Arrival and Settling In** 

- **Morning**: Arrive at the airport. Welcome to your adventure! Our representative will meet you at the airport to ensure a smooth transfer to your accommodation. 
- **Afternoon**: Check into your hotel and take some time to relax and refresh. 
- **Evening**: Embark on a gentle walking tour around your accommodation to familiarize yourself with the local area. Discover nearby dining options for a delightful first meal. 

**Day 2 – A Day of Culture and Nature** 

- **Morning**: Start your day at Imperial College, one of the world's leading institutions. Enjoy a guided campus tour. 
- **Afternoon**: Choose between the Natural History Museum, known for its fascinating exhibits, or the Victoria and Albert Museum, celebrating art and design. Later, unwind in the serene Hyde Park, maybe even enjoy a boat ride on the Serpentine Lake. 
- **Evening**: Explore the local cuisine. We recommend trying a traditional British pub for dinner.

**Additional Services:** 

- **Concierge Service**: Throughout your stay, our concierge service is available to assist with restaurant reservations, ticket bookings, transportation, and any special requests to enhance your experience. 
- **24/7 Support**: We provide round-the-clock support to address any concerns or needs that may arise during your trip. 

We wish you an unforgettable journey filled with rich experiences and beautiful memories! 

### Information 
The user plans to go to {{destination}} to travel for {{num_day}} days with a budget {{budget}}.
```
