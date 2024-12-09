
+++
title = "LLM Serving"
date = "2024-12-09"
tags = ["sys", "ai", "llm"]
description = "本文总结LLM serving的计算形态和优化机会。"
showFullContent = false
+++

LLM推理有别于小模型推理，更加memory-bound，输入输出也有可以利用的独特模式，自然涌现出系统层面可优化的点。我将这些系统层面的优化机会粗略分为batching优化、sampling优化、模型压缩3类。

其中batching优化负责输入端将GPU等加速器的计算单元喂的更饱和、更好地利用SRAM locality[^5]，sampling优化[^6]负责输出端更快速拿到LLM的结果，模型压缩负责让大模型不要那么大。

## LLM Serving的计算形态
LLM serving相比此前主流的深度模型serving，有一些独有的特性和约束：
- 一次LLM调用中很多时间耗费在加载模型参数上（HBM->SRAM）。
    - batching可以让一次参数加载负责多个seq的推理，显著减少了总体的模型参数加载开销。
- Seq2Seq：变长输入、变长输出。
    - 因此batching机制必须适应batch size，seq len皆为变量。
- 自回归（通过若干次迭代才能处理一个请求），且为每个对话维护一个跨迭代的KV。
    - 纯粹无状态的方法需每次重新计算所有KV，开销会大得无法接受，因此必须基于KV caching做incremental decoding。
    - RNN也是自回归，transformer相比RNN，还有一个不同之处是每次迭代KV cache都会变大。
    - 非自回归模型往往以request为粒度做batching，对自回归场景并不适用，后者明显更适合以iteration为粒度。
- GPT不同于原版transformer，是decoder-only的。
    - 此前encoder-only和encoder-decoder架构的变长batching padding策略不适用于LLM。
- GPT多了一个前置的计算密集的prefill阶段（也叫infill），用于处理prompt。
    - prefill阶段和增量迭代阶段的序列不好batching（计算要完全一样才好batching）。
    - 这个阶段消化prompt，考虑到很多LLM应用都有非常巨大、内容也差不多的prompt，prefill阶段很多计算都是重复的。
- GPT在生成token概率分布后，还有一个后置的sampling/decoding阶段：基于概率密度从vocab中选择token。
    - 考虑到vocab比较大，各种采样算法的计算量都很小，这个过程显然是访存密集的。
    - token选择完成后，当前迭代选中的token还需要再作为下次迭代的forward pass输入参数。
    - 若seq[A~A+5]在此前的迭代中已经进行了采样，在新一轮迭代中除了需要对seq[A+6]采样之外，仍然要对seq[A~A+5]再算一遍。
    - 已处理tokens的sampling/decoding结果可以cache下来。
    - 很多LLM应用会要求结构化输出，输出的格式比较固定。
- Transformer的attention算子的input张量形状和已处理tokens长度有关。
    - 不同序列，序列长度不同，attn计算的输入形状不统一，就不好做batching。
    - 好在attn计算做不做batching影响不大，因为attn计算并不涉及任何模型权重，也就不存在一次参数加载重用于多序列推理的加速效应。
- LLM在GPU上跑的一个关键瓶颈是将数据（请求+参数）加载进HBM的开销。LLM serving吞吐明显会受限于batch size——即一次性能放入多少数据而不至于爆显存。
    - 在简单的静态batching实现中，seq_len * nr_seqs + 模型大小决定了显存占用。seq_len假设定的比较高，又用不完，那也会让nr_seqs缩小很多。
    - 显存如此紧张，自然使量化、剪枝等模型压缩技术在LLM serving中变得尤为重要，比如awq, gptq, gguf, smooth quant。

## Continuous Batching
上文提及LLM做batching好处虽多，困难更多，并不存在特别简单、显而易见的GPT特供batching机制。

Orca[^1]首先为GPT模型的batching提出了一个完整的解决方案，在迭代层面进行调度，并通过一个必要的selective batching机制剔除不能做batching的操作（若不做剔除，两个请求整体能做batching的几率可以忽略不计），这些操作虽然不能做batching，但对整体性能提升的影响很小。

![orca](https://cmbbq.github.io/img/orca.png)

## Paged Attention
Orca方案并未考虑KV cache的HBM占用，默认预分配max_seq_len。

但monolithic KV cache导致HBM碎片化，为每个seq预分配巨大内存进而导致并发不足也是真实瓶颈所在，vLLM中就对此进行了优化，提出了Paged Attention[^2]，一种近似页表的分块kv cache技术：
- 在prefill阶段允许kv cache在非连续内存中以页的方式组织起来，因此不必为max_seq_len提前分配内存，运行时再分配就好。
- 大多数seq显然不会触及max_seq_len，paged attention因此节省了大量内存，也就允许batch size大大提高。
- vLLM的实现中并未采纳Orca的selective batching，主要是因为它的paged attention算子是自己写的cuda，可以与非attn算子一起兼容batching。vLLM将prefill和decoding分开做batching，整体上就不需要实现selective batching这么麻烦的机制了。
    - 但这种做法也阻止了prefill和decoding step的融合。如果某个prompt过长，prefill开销太大，确实会出现block后续所有decoding batch的情形。

![block_table](https://cmbbq.github.io/img/block_table.png)

详见[Paged Attention](https://cmbbq.github.io/posts/paged-attention)。

## Dynamic SplitFuse
DeepSpeed-FastGen[^3]中提出了SplitFuse，也是Continuous Batching的一个演化版本。思路是切分长prompt请求成若干个小的step，这些小的step开销较低，可填充调度缝隙，同时还保证prefill(prompt generation)和decode(token generation)的steps开销一致，就可以确保不存在大小不同的workload，取得一定的吞吐提升，但最主要的优势是能稳住tail-latency，在线服务场景下下限更高。

![split_fuse](https://cmbbq.github.io/img/split_fuse.png)

## Quantization and Pruning
由于Nvidia架构上天然偏向图形负载，显存天然就给的不足，各种量化、剪枝技术除了降低计算量外，在LLM serving方向上还起到了关键性的降低显存占用的效果。

详见[Quantization and Pruning](https://cmbbq.github.io/posts/quantization-and-pruning)

## Radix Attention
SGLang采用了Radix attention[^4]技术，将common prefix的KV以radix tree的形式保留下来，使kv cache的生命周期不局限于一次请求，而是真正构成跨多次请求的LRU cache，适应prompt巨大且大多有相同前缀的实际应用场景。

## Flash Attention
Continuous batching提升了非attn操作的SRAM locality，针对kv计算，Flash Attention[^8]则令attn计算内层循环fit in SRAM。

![fast_att](https://cmbbq.github.io/img/fast_attention.png)

详见[Flash Attention](https://cmbbq.github.io/posts/flash-attention)。

## Speculative Decoding
Speculative decoding[^7]的思路是选用tokenizer相同，大小不同的两个模型。假设大模型的latency大体上是小模型的N倍。小模型输出N个token的时间内，大模型把这N个token拿过来append到seq上形成input，再输出1个token，总计生成N+1个token。基于greedy decoding对这N+1个token进行采样，采样结果如果和小模型的结果match，就直接用了，不match就停下，在停下的地方把原本的小模型token改成大模型采样结果对应的token。整个过程中，大模型实际上只需要一个forward pass，运气好就能一下子输出N+1个token，运气差就输出1个token。

![speculative_decoding](https://cmbbq.github.io/img/speculative_decoding.png)


## Structured Decoding
SGLang基于一个压缩有限状态机实现了structured decoding[^4]，用于对特定结构化输出（比如支持regex的JSON模板）进行加速，一次性decode多个token。假设这个结构化输出的JSON模板中总是有一个key是"top5 candidate"，那就可以把"top5 candidate"这个multi-token词组当成一个token一轮迭代处理掉。

![structured_decoding](https://cmbbq.github.io/img/structured_decoding.png)


[^1] Gyeong-In Yu and Joo Seong Jeong. Orca: A Distributed Serving System for Transformer-Based Generative Models. OSDI 22. [[pdf]](https://www.usenix.org/system/files/osdi22-yu.pdf)
[^2]: W. Kwon, et al. Efficient Memory Management for Large Language Model Serving with PagedAttention. [[pdf]](https://arxiv.org/pdf/2309.06180)
[^3]: C. Holmes, et al. DeepSpeed-FastGen: High-throughput Text Generation for LLMs via MII and DeepSpeed-Inference. [[pdf]](https://arxiv.org/pdf/2401.08671)
[^4]: L. Zheng, et al. SGLang: Efficient Execution of Structured Language Model Programs. [[pdf]](https://arxiv.org/pdf/2312.07104)
[^5]: GPU/NPU/TPU等加速器需将模型参数从off-chip memory加载到on-chip SRAM才能进行底层硬件算子的计算，对较大的模型，这种加载开销往往才是瓶颈所在。因此batching不仅仅能提升加速器计算单元的利用率，还能通过一份模型参数在多个请求中重用，更好地利用SRAM locality。
[^6]: 在LLM推理语境下，sampling和decoding经常混用：有时都是指的基于density做token-selection的过程，有时decoding是指整个decoder-only transformer的inference过程。
[^7]: Y. Leviathan, et al. Fast Inference from Transformers via Speculative Decoding. [[pdf]](https://arxiv.org/pdf/2211.17192)
[^8]: T. Dao, et al. FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness. [[pdf]](https://arxiv.org/pdf/2205.14135)
