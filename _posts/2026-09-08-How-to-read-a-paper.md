---
title: How to Read a Paper｜三轮阅读法
date: 2026-09-08 12:00:00 +0800
tags: [论文阅读, 方法论, 学术]
categories: [学习]
---

> 原文：*How to Read a Paper*，S. Keshav，David R. Cheriton School of Computer Science, University of Waterloo，Waterloo, ON, Canada，keshav@uwaterloo.ca。版本：February 17, 2016。
>
> 本文只保留三轮阅读法的核心段落，背景、文献综述、致谢等部分做极简处理。

## 核心总述

**原文**

> The key idea is that you should read the paper in up to three passes, instead of starting at the beginning and plowing your way to the end. Each pass accomplishes specific goals and builds upon the previous pass: The first pass gives you a general idea about the paper. The second pass lets you grasp the paper's content, but not its details. The third pass helps you understand the paper in depth.


> 核心思想：不要从开头硬读到结尾，最多分三轮阅读。每一轮有明确目标，层层递进：第一轮建立整体概念；第二轮掌握主体内容，不纠结细节；第三轮实现深度理解。

---

## 第一轮：快速概览 The first pass（5-10 min）

**原文**

> The first pass is a quick scan to get a bird's-eye view of the paper. You can also decide whether you need to do any more passes. This pass should take about five to ten minutes and consists of the following steps:
>
> 1. Carefully read the title, abstract, and introduction
> 2. Read the section and sub-section headings, but ignore everything else
> 3. Glance at the mathematical content (if any) to determine the underlying theoretical foundations
> 4. Read the conclusions
> 5. Glance over the references, mentally ticking off the ones you've already read
>
> At the end of the first pass, you should be able to answer the five Cs:
>
> 1. Category: What type of paper is this? A measurement paper? An analysis of an existing system? A description of a research prototype?
> 2. Context: Which other papers is it related to? Which theoretical bases were used to analyze the problem?
> 3. Correctness: Do the assumptions appear to be valid?
> 4. Contributions: What are the paper's main contributions?
> 5. Clarity: Is the paper well written?


第一轮做快速扫视，获得全局视角，并决定要不要继续读下去。操作步骤：

1. 仔细读标题、摘要、引言
2. 只看章节与子章节标题，跳过正文
3. 扫一眼数学内容，把握底层理论
4. 阅读结论
5. 浏览参考文献，标记已读过的文献

读完回答 **5 个 C**：

- Category（类别）：论文属于哪一类？
- Context（背景）：关联哪些工作，使用什么理论？
- Correctness（正确性）：假设是否合理？
- Contributions（贡献）：核心创新是什么？
- Clarity（清晰度）：文章写作质量如何？

> 判断：不相关、看不懂、假设不合理 → 直接停止阅读。

---

## 第二轮：掌握主体内容 The second pass（约 1 小时）

**原文**

> In the second pass, read the paper with greater care, but ignore details such as proofs. It helps to jot down the key points, or to make comments in the margins, as you read.
>
> 1. Look carefully at the figures, diagrams and other illustrations in the paper. Pay special attention to graphs. Are the axes properly labeled? Are results shown with error bars, so that conclusions are statistically significant?
> 2. Remember to mark relevant unread references for further reading.
>
> After this pass, you should be able to grasp the content of the paper. You should be able to summarize the main thrust of the paper, with supporting evidence, to someone else.
>
> Sometimes you won't understand a paper even at the end of the second pass… You can now choose to: (a) set the paper aside, (b) return to the paper later, perhaps after reading background material or (c) persevere and go on to the third pass.


第二轮更加仔细阅读，但跳过证明等复杂细节，随手记笔记、记下不懂的术语与疑问。重点：

1. 仔细看图表，检查坐标轴、误差棒，判断实验严谨程度
2. 标记需要后续阅读的参考文献

读完标准：可以向他人复述论文核心观点与证据。

读完仍然看不懂，三选一：

1. 搁置论文
2. 补背景知识之后重读
3. 继续进入第三轮

---

## 第三轮：深度吃透 The third pass（数小时）

**原文**

> To fully understand a paper, particularly if you are a reviewer, requires a third pass. The key to the third pass is to attempt to virtually re-implement the paper: that is, making the same assumptions as the authors, re-create the work. By comparing this re-creation with the actual paper, you can easily identify not only a paper's innovations, but also its hidden failings and assumptions.
>
> This pass requires great attention to detail. You should identify and challenge every assumption in every statement. Moreover, you should think about how you yourself would present a particular idea.
>
> At the end of this pass, you should be able to reconstruct the entire structure of the paper from memory, as well as be able to identify its strong and weak points. In particular, you should be able to pinpoint implicit assumptions, missing citations to relevant work, and potential issues with experimental or analytical techniques.


适合审稿、课题核心文献，需要完全读懂。核心技巧：**在脑海虚拟复现论文**——采用和作者相同的假设，重新推演一遍研究，再与原文对比。

- 质疑文中每一条前提假设
- 换位思考：如果我来做，我会如何表达这个想法
- 记录未来研究思路

完成标准：凭记忆还原整篇论文结构；明确指出论文优缺点、隐含假设、缺引文献、实验方法潜在缺陷。

---

## 实践小结

| 轮次 | 耗时 | 目标 | 是否每篇都要做 |
| --- | --- | --- | --- |
| 第一轮 | 5-10 min | 筛选论文，建立全局概览 | ✅ 所有论文必做 |
| 第二轮 | ~1 h | 理解核心内容，跳过细节 | ✅ 感兴趣的论文 |
| 第三轮 | 1h+ | 深度批判、虚拟复现 | ❌ 仅核心论文、审稿场景 |

> 不是每一篇论文都需要读到第三轮，大部分文献止步第一轮或第二轮即可。
