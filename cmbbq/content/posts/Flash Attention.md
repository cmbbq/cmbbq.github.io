
+++
title = "Flash Attention"
date = "2024-07-22"
tags = ["sys", "ai", "llm"]
description = "Flash Attention，一言以蔽之：tiling + selective gradient checkpointing。"
showFullContent = false
+++

由于`self-attention`的时、空复杂度都是序列长的平方，长序列LLM和高分辨率ViT都是非常吃内存的。

此前针对`self-attention`的优化大多是近似计算，其核心是优化FLOP，把理论上的时间复杂度降低到O(N)，但这并不能有效加速`self-attention`，因为该操作（以及transformer中多数操作）的实际瓶颈在访存——更准确地说是HBM和SRAM之间的IO。

`FlashAttention`的原理就是基于`tiling`，确保内层循环计算fit in SRAM，减少了HBM和SRAM之间的IO频次，因而真切有效地提升了transformer性能，解锁了更长的context，迅速在各种高性能框架中得到应用。

## 传统的Self-Attention实现
Attention Layer具有将局部信息和张量中较远位置的信息结合起来的能力——通过为所得张量的每个组件与输入张量的每个组件计算注意力得分，不受局部性约束地，在整个张量范围内对特征进行平均。

给定$N^Q\times D^{QK}$维queries张量$Q$，$N^{KV}\times D^{QK}$维keys张量$K$，$N^{KV}\times D^V$维values张量$V$，通过```Attention操作```$att(K,Q,V)$计算得到$N^Q\times D^V$维的张量Y：

$$Y = att(K,Q,V) = \underbrace{softargmax(\frac{QK^T}{\frac{1}{\sqrt{D^{QK}}}})}_A V$$

整个过程分两步，第一步先计算每个query index $q$和每个key index $k$的注意力得分，即queries和keys点乘后的```softargmax```结果：$A_{q,k} = \frac{exp(\frac{1}{\sqrt{D^{QK}}} Q_q \cdot K_k ) }{\sum_l exp(\frac{1}{\sqrt{D^{QK}}} Q_q \cdot K_l)}$，其中$\frac{1}{\sqrt{D^{QK}}}$是一个缩放参数，用于保证取值范围在$D^{QK}$变大时大体不变。

![att](https://cmbbq.github.io/img/attention.png)

得到注意力得分$A_{q,k}$后，再进行第二步计算：$Y_q = \sum_k A_{q,k}V_k$。注意力得分即queries和keys之间的匹配程度，匹配程度越高，该权重就越高。如果某个query和某个key匹配度达到极限，注意力得分接近1，则直接拿到这个key对应的value。如果和好几个key都有中等水准的匹配，则按注意力得分做加权平均。

## FlashAttention的IO优化
`FlashAttention`注意到事实上并不需要完整的输入K、V、Q，可以分批读入、分批计算、分批把结果写回O，即所谓`tiling`：
- 在外层循环，遍历K、V矩阵。每次只需加载一个block的 $K^T$ 和 $V$ 到片上SRAM中。
- 在内层循环，遍历Q的各个block，加载到SRAM中，进行 $att(K,Q,V)$ 计算，把局部结果写回HBM上 $N\times d$ 的结果矩阵[^3]。
- 对softmax归一化因子做相应调整，即可保证最后加起来得到的最终结果和标准实现等价，具体代数见[^1]附录。
- 设置K、V的block size为 $\lceil \frac{M}{4d} \rceil$，Q、O的block size为$min(\lceil \frac{M}{4d} \rceil, d)$[^4]。

![fast_att](https://cmbbq.github.io/img/fast_attention.png)

此外，对于训练负载来说，`FlashAttention`还在反向传播中复用前馈过程暂存的softmax归一化因子$\frac{1}{\sqrt{D^{QK}}}$，这也比从HBM读$N\times N$的巨大attention中间矩阵要快得多。这可被视作selective gradient checkpointing。

## FlashAttention2: 改进并行和工作切分
相比GEMM，`FlashAttention`只达到了25~40%理论FLOPs/s，可优化空间巨大。`FlashAttention2`[^2]在原版基础上做了并行化和工作切分优化。
- 减少非矩阵乘算子，因为GPU矩阵乘是高度优化的，其他算子与之差距非常大。
    - 避免循环中算O时每次都rescale，而是在算最终结果时施加softmax归一化因子。
    - 对反向传播时保持的状态进行了精简。
- 在不同线程块[^8]上进行并行计算，从而充分利用GPU资源。
    - 原版一个线程块处理一个head/一个batch，每个线程块跑在一个SM上。不过在长序列场景，head数目和batch size可能都偏小，导致二者相乘后都未必能打满A100的128个SM。
    - 显而易见可以并行的部分是外层循环，可以随便调度到不同线程块上，相互之间完全没有通信需求。
    - 反向传播时并行化外层循环也只有dQ更新时需要简单的通信/同步，这是一个顺序不重要的加法，atomic add足以解决。

既然有了线程块，就要考虑在每个线程块内，不同warps之间如何进行工作的切分。
![work_part](https://cmbbq.github.io/img/work_part.png)

如上图所示，`FlashAttention`的切分方式导致内层循环中各个warp都需要把结果写到共享内存且做一个同步加，存在一定的通信开销，而`FlashAttention2`的切分方式可以保证warp之间完全没有通信需求。


[^1]: T. Dao, et al. FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness. [[pdf]](https://arxiv.org/pdf/2205.14135)
[^2]: Tri Dao. FlashAttention-2: Faster Attention with Better Parallelism and Work Partitioning. [[pdf]](https://arxiv.org/pdf/2307.08691)
[^3]: 其中，d为head dimension。N为序列长度。$N \gg d$。GPT2中 $N=1024，d=64$。
[^4]: 其中，M为SRAM大小。
[^5]: M. Milakov, N. Gimelshein. Online Normalizer Calculation for Softmax. [[pdf]](https://arxiv.org/pdf/1805.02867)
[^6]: Markus N. Rabe, Charles Staats. Self-attention Does not Need $O(n^2)$ Memory. [[pdf]](https://arxiv.org/pdf/2112.05682)
[^7]: W. Kwon, et al. Efficient Memory Management for Large Language Model Serving with PagedAttention. [[pdf]](https://arxiv.org/pdf/2309.06180)
[^8]：多个线程块，一个线程块(thread block)内的多个warp可分时复用同一个SM。