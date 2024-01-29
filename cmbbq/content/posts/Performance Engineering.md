+++
title = "Practical Performance Engineering of Memory-Bound Systems"
date = "2024-01-29"
tags = ["sys", "en"]
description = "Optimization can be divided into work reduction and hardware enablement."
showFullContent = false
+++

[WIP]

# Work Reduction vs Hardware Enablement
> Optimization can be divided into work reduction and hardware enablement. 

Work, in terms of computer programming, is basically a gross measure of how much stuff the program needs to do. 

The idea of work optimization is to reduce the amount of stuff the program needs to do. Commonly employed techniques include: approximation, tail-recursion elimination, coarsening/refining recursion, inlining, loop fusion, loop unrolling, hoisting, short-circuiting, common-subexpression elimination, compile-time initialization and evaluation, exploiting sparsity, caching, pre-computation, and bit hacks. 

While reducing work undoubtedly serves as an essential heuristic for reducing overall running time, it's not the sole determinant of the running time; it doesn't capture the whole picture of computer programming, leaving the intricate nature of computer hardware unaddressed. 

To thoroughly investigate architectural improvements that unlock hardware potential, we must delve into numerous aspects in hardware and micro-architecture: the ISA, pipeline stages, superscalar processing, out-of-order execution, paging, caching, vectorization, speculation, hardware prefetching, branch prediction, etc. 

> Historically computer architecture leverages either locality or parallelism to enhance performance. 

To exploit locality, the memory hierarchy(registers->L1/L2/L3 caches->local DRAM->remote DRAM->PMem->SSD) was made deeper for hiding performance issue; hardware prefetchers and branch predictors were used to predict immediate accesses and move data or instructions closer to processors. As programmers, we are working on one layer obove the computer architecture layer, what we can do is to write NUMA-aware, cache-aligned, and perhaps vectorized code with regular data access patterns and proper software pre-fetching.

To exploit parallelism, superscalar out-of-order pipelines, breaking ops into micro-ops, vector hardware, multi-core were invented. Correspondingly, we need to keep all these things busy by using techniques like bit tricks, ILP(Instruction-level parallelism), AVX/SSE, AMX, multithread/multi-process programming and offloading to accelerators like DPUs or GPUs.  

Let's delve deeper into ILP, as it is more connected with the processor μ-arch - aspects such as superscalar processing, pipeline stages, out-of-order execution, data bypassing, register renaming, and so on. To exploit it programmatically, we can use separate functional units, write likely/unlikely hints for branch prediction, and break dependency in the data-flow graph beforehand to reduce data hazards.  

# Lessons Learnt from Prior Work
[understand your program. measure its Arithmetic Intensity]


# Interesting Heuristics for Future Work
## Employing Quantitative Approaches 
[off-cpu flame graph] 

## Playing with DRAM as if We were Handling Slow Devices
As `DRAM` access has become the de-facto bottleneck for many of today's datacenter applications, an intuitive idea to optimize these applications would be shamelessly copying prior wisdom on dealing with slow devices like `PMem`s or `SSD`s. 

`PMem`s offer only $\frac{1}{14}$~$\frac{1}{3}$ of DRAMs' bandwidth. `SSD`s are typically 1~2 orders of magnitude slower than DRAMs. To overcome their slowness, many data structures were crafted specifically for them.

`Dashtable`[Lu et al., 2020][^4], for example, was targeting scalable hasing on the once fashionable `PMem`s. Hasing on `PMem`s sounds somewhat niche. But in reality, if any being could survive `PMem`'s hellish bandwidth, you can definitely count on it for arbitrary memory-bound application. In 2022, Intel killed off its `PMem` business[^5]. `PMem` died. Yet the `Dash` methodology still thrives. Today we use it in regular `DRAM` setups. Check this out, the [DragonFly](https://github.com/dragonflydb/dragonfly) project. It's based on `Dash` and claims to be the modern replacement for Redis and Memcached, delivering 25x more throughput. 

The core idea behind `Dashtable` is trading space for time, or more specifically, each bucket paying a little bit extra space in metadata to buy (1)faster bucket probing with fingerprints of keys and (2)lightweight optimistic concurrency control with version locks, (3)stashing overflow records, which helps to alleviate segment splits caused by unbalanced bucket loads. 

Another noticeable development is that modern-day system language `Rust`, has opted for `B-tree`s over `Red-black tree`s for its ordered maps. The 50+ years old `B-tree`, once targeting disk storage, might find its place in today's memory-bound applications. That is interesting because the `Red-black tree`s was deemed memory-efficient and is still widely in use as the default implementation of `C++`'s `std::map`. Yet `B-tree` somehow beats `Red-black tree` on modern servers.

So what happened to modern hardware? Well, it mostly involves the memory wall problem, which refers to the increasing gap between processor and memory speed. For decades, the memory wall problem has never been resolved, but only mitigated by introducing deeper and deeper memory hierarchies. In today's architectures, L3 become much slower than L1/L2, and `DRAM`s should be labeled as dangerously slow as disks were in the 1980s perspective.

[^1]: [Tuning Guides for Intel® Xeon® Scalable Processor-Based Systems](https://www.intel.com/content/www/us/en/developer/articles/guide/xeon-performance-tuning-and-solution-guides.html#gs.3h82qm)
[^2]: [Finding arithmetic intensity of application](https://community.intel.com/t5/Software-Tuning-Performance/Finding-arithmetic-intensity-of-application/m-p/1093205#M5700)
[^3]: [MIT 6.172 Performance Engineering of Software Systems](https://ocw.mit.edu/courses/6-172-performance-engineering-of-software-systems-fall-2018/pages/lecture-slides/)
[^4]: B. Lu, X. Hao, T. Wang, and E. Lo. Dash: Scalable Hashing on Persistent Memory. CoRR abs/2003.07302, 2020. [[pdf]](https://arxiv.org/pdf/2003.07302.pdf).
[^5]: [Intel kills off Optane Memory, writes off $559 million inventory](https://www.datacenterdynamics.com/en/news/intel-kills-off-optane-memory-writes-off-559-million-inventory/)

