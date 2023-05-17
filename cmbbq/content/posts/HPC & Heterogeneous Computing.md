+++
title = "A Decade of CPU vs GPU Tussle"
date = "2022-09-25"
tags = ["ai", "sys"]
description = "黄氏定律是否成立？加速计算十年之争是否已尘埃落定？本文回顾了过去十年GPU和CPU的价格-性能趋势，试图回答『GPU和CPU方法的边界何在？』这一AI服务架构工作的永恒话题。"
showFullContent = false
+++

[WIP]

## GPU和CPU方法的边界何在？
做AI应用的架构工作，遇到新的计算密集任务的第一问往往是：这应该上GPU，还是NUMA物理机？

这个问题可以归约为[On the Limits of GPU Acceleration(2010)](https://www.usenix.org/legacy/event/hotpar10/tech/full_papers/main.pdf)中提出的问题："大致相同能耗的前提下，能和不能用GPU有效加速计算的边界在哪？" Vuduc的这个研究分析了3种具有代表性复杂行为和访存不规则性的计算：稀疏线性系统的迭代求解器、稀疏矩阵的乔列斯基分解、快速多极子算法，得出的结论是——大致地说，良好优化的GPU实现在相同能耗下和良好优化的CPU实现在性能上是相仿的。可见当年用GPGPU加速还是一个普遍得不偿失的工作，毕竟要付出剧烈变更编程模型的代价。

## GPU和CPU的价格-性能趋势
上述结论在2010年成立，如今13年过去了，GPU固然飞速发展（无论是硬件、软件，还是生态、应用、资金投入），但GPU发展速度是否已经甩开CPU了呢？

GPU的发展速度毫无疑问是领先CPU的，[Vnatoli的文章](https://www.nextplatform.com/2019/07/10/a-decade-of-accelerated-computing-augurs-well-for-gpus/)。。。。

摩尔定律是两年翻倍，而[黄氏定律](https://en.wikipedia.org/wiki/Huang%27s_law)则是宣称通过软硬件协同能达到1.08年翻倍。

甚至如果我们考虑成本因素，根据[经验数据](https://epochai.org/blog/trends-in-gpu-price-performance)，GPU FLOP/s每刀的增长速度是和CPU相仿的。这和2023年工业界的经验是吻合的——CPU在大多数应用语境下依然是成本更低的选择。

![Emprical GPU/CPU FLOP/s per dollar1](https://cmbbq.github.io/img/4.png)
![Emprical GPU/CPU FLOP/s per dollar2](https://cmbbq.github.io/img/5.png)
![Emprical GPU/CPU FLOP/s per dollar3](https://cmbbq.github.io/img/6.png)

## 