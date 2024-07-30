
+++
title = "Paged Attention"
date = "2024-07-30"
tags = ["sys", "ai"]
description = "Paged Attention：提升同时处理多请求的显存利用率和吞吐。"
showFullContent = false
+++

在LLM serving场景，通过恰当的batching积攒足够多的请求，可提升LLM吞吐。但每个请求对应的KV Cache非常巨大——在原始实现中，KV Cache需要为`max_tokens`预留内存，但每个请求实际上携带的tokens数目普遍远小于`max_tokens`，造成大量内存浪费和反复的动态内存分配。

`PagedAttention`[^1]的目标就是消除这种内存浪费、在请求内部和请求之间灵活共享一些KV cache。vLLM通过`PagedAttention`达到此前sota的`FasterTransformer`和`Orca`的2~4倍吞吐。

此前，朴素的KV Cache实现如下图所示，简短的prompt和并不长的当前iteration只占据了7+4个slots，剩余2038个slots不得不预留在内存里以支撑最大序列长达2048的承诺（采样结束才能知道实际tokens数目），在这之后才能是下一个请求的slots，中间预留的部分就完全浪费掉了。

![naive_kv_cache](https://cmbbq.github.io/img/naive_kv_cache.png)

`PagedAttention`把连续的kv存在不连续的内存空间上，借用类似页表的机制（引入一个block_table）规避内存碎片化问题。具体来讲，`PagedAttention`把KV cache分区成若干个K blocks和V blocks，每个K/V block容纳固定数量tokens所对应的K/V向量，因此attention计算也被转化为blockwise计算。这和`FlashAttention`有点像，不过应用的尺度不同，前者是为了大规模serving时克服碎片化、按需取用，后者是为了单次self-attention计算能全部fit in SRAM。

![block_table](https://cmbbq.github.io/img/block_table.png)

这个物理kv blocks显然是支持多个请求复用的。如下图所示，为每个请求维护一个小的block table即可。

![vllm_two_requests](https://cmbbq.github.io/img/vllm_two_requests.png)


[^1]: W. Kwon, et al. Efficient Memory Management for Large Language Model Serving with PagedAttention. [[pdf]](https://arxiv.org/pdf/2309.06180)

