+++
title = "Practical Performance Engineering of Memory-Bound Systems"
date = "2024-01-24"
tags = ["sys"]
description = "WIP"
showFullContent = false
+++

WIP

# Performance Engineering Taught in Schools
[summarize mit 6.172]


# Lessons Learnt from Prior Work
[understand your program. measure its Arithmetic Intensity]


# Interesting Heuristics for Future Work
## Adopt Quantitative Approaches 
[off-cpu flame graph] 


## Playing with DRAM as if We were Handling Slow Devices
As `DRAM` access has become the de-facto bottleneck for many of today's datacenter applications, an intuitive idea to optimize these applications would be shamelessly copying prior wisdom on dealing with slow devices like `PMem`s or `SSD`s. 

`PMem`s offer only $\frac{1}{14}$~$\frac{1}{3}$ of DRAMs' bandwidth. `SSD`s are typically 1~2 orders of magnitude slower than DRAMs. To overcome their slowness, many data structures were crafted specifically for them.

The well-known `B-tree`, for example, was originally targeting disk storage. `Red-black tree`, on the other hand, is known as C++ ```std::map```'s default implementation and was indeed efficient from a 1990s era point of view. But today's `DRAM` might have become too slow for `Red-black tree`s. Modern day system language, Rust, opts for `B-tree`s over `Red-black tree`s for ordered maps due to their cache-friendliness and better performance on modern hardware with much deeper memory hierarchies. 

The `Dashtable`[Lu et al., 2020][^4], as another example, was targeting the once fashionable `PMem`s. Hasing on `PMem` sounds somewhat niche. But in reality, any memory-constrained application can benefit from it. In 2022, Intel killed off its `PMem` business[^5]. `PMem` died. Yet the `Dash` methodology, which survived `PMem`'s hellish bandwidth, still thrives, shifting to regular `DRAM` applications, as we can see in projects like [DragonFly](https://github.com/dragonflydb/dragonfly).

`Dashtable`, the proposed scalable hashtable, evolves from `extendible hashing`. `Extendible hashing` is a hash system that uses $N$ bits of hashed values to look up buckets in a trie-structured `directory`. A `directory` with `global depth` $N$ can hold $2^N$ buckets. Each bucket also has a `local depth` $M = \mathrm{Len}(\mathrm{Hash}(key)) - N$ and can hold up to $2^M$ keys. Additionally, the lightweight concurrency control in `Dash` naturally scales well on today's `many-core` architectures. 



[^1]: [Tuning Guides for Intel® Xeon® Scalable Processor-Based Systems](https://www.intel.com/content/www/us/en/developer/articles/guide/xeon-performance-tuning-and-solution-guides.html#gs.3h82qm)
[^2]: [Finding arithmetic intensity of application](https://community.intel.com/t5/Software-Tuning-Performance/Finding-arithmetic-intensity-of-application/m-p/1093205#M5700)
[^3]: [MIT 6.172 Performance Engineering of Software Systems](https://ocw.mit.edu/courses/6-172-performance-engineering-of-software-systems-fall-2018/pages/lecture-slides/)
[^4]: B. Lu, X. Hao, T. Wang, and E. Lo. Dash: Scalable Hashing on Persistent Memory. CoRR abs/2003.07302, 2020. [[pdf]](https://arxiv.org/pdf/2003.07302.pdf).
[^5]: [Intel kills off Optane Memory, writes off $559 million inventory](https://www.datacenterdynamics.com/en/news/intel-kills-off-optane-memory-writes-off-559-million-inventory/)