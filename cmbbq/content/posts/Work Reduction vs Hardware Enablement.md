
+++
title = "Work Reduction vs Hardware Enablement"
date = "2024-02-08"
tags = ["sys", "en", "perf"]
description = "Optimization can be divided into work reduction and hardware enablement."
showFullContent = false
+++

> Optimization can be divided into work reduction and hardware enablement. 

Work, in terms of computer programming, is basically a gross measure of how much stuff the program needs to do. 

The idea of work optimization is to reduce the amount of stuff the program needs to do. Commonly employed techniques include: `approximation`, `tail-recursion elimination`, `coarsening/refining recursion`, `inlining`, `loop fusion`, `loop unrolling`, `hoisting`, `short-circuiting`, `common-subexpression elimination`, `compile-time initialization`, `compile-time evaluation`, `exploiting sparsity`, `caching`, `pre-computation`, and `bit hacks`. 

While reducing work undoubtedly serves as an essential heuristic for reducing overall running time, it's not the sole determinant of the running time; it doesn't capture the whole picture of computer programming, leaving the intricate nature of computer hardware unaddressed. 

To thoroughly investigate architectural improvements that unlock hardware potential, we must delve into numerous aspects in hardware and micro-architecture: `the ISA`, `pipeline stages`, `superscalar processing`, `out-of-order execution`, `paging`, `caching`, `vectorization`, `speculation`, `hardware prefetching`, `branch prediction`, etc. 

> Historically computer architecture leverages either locality or parallelism to enhance performance. 

To exploit locality, the memory hierarchy(`registers`->`L1/L2/L3 caches`->`local DRAM`->`remote DRAM`->`PMem`->`SSD`) was made deeper for hiding performance issue; hardware prefetchers and branch predictors were used to predict immediate accesses and move data or instructions closer to processors. As programmers, we are working on one layer obove the computer architecture layer, what we can do is to write NUMA-aware, cache-aligned, and perhaps vectorized code with regular data access patterns and proper software pre-fetching.

To exploit parallelism, superscalar out-of-order pipelines with micro-ops, vector hardware, multi-core were introduced. Correspondingly, we need to keep all these things busy by using techniques like `bit tricks`, `ILP`(Instruction-level parallelism), `AVX`/`SSE`, `AMX`, multithread/multi-process programming and offloading to accelerators like `DPU`s or `GPU`s.  

Let's explore `ILP` further, as it is more connected with the μ-arch designs inside a processor, such as out-of-order execution, data bypassing, register renaming, and so on. To exploit CPU μ-arch programmatically, we can (1)use separate functional units, (2)write likely/unlikely hints for branch prediction, and (3)break dependency in the data-flow graph beforehand to reduce data hazards.  
