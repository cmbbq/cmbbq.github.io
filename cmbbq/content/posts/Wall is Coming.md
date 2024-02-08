+++
title = "Wall is Coming"
date = "2024-02-07"
tags = ["sys", "en", "perf"]
description = "The memory wall is coming."
showFullContent = false
+++

The memory wall is coming. Given that DRAM access has become the de-facto bottleneck for many of today's datacenter applications, it seems logical to optimize these applications by shamelessly copying prior wisdom on dealing with slow peripherals like `PMem`s or `SSD`s. 

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

