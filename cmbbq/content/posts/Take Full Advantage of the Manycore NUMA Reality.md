+++
title = "Take Full Advantage of the Manycore NUMA Reality"
date = "2023-05-13"
tags = ["sys"]
description = "当一个计算密集服务成为互联网规模的基础设施时，高性能计算的需求就变得迫切，因为性能意味着时间、能源和金钱。本文主要讨论现代商用硬件上如何通过share-nothing架构最大化众核+深内存层次结构时代的访存密集型CPU-bound应用的性能。"
showFullContent = false
+++

[WIP]

当一个计算密集服务成为互联网规模的基础设施时，高性能计算(HPC)的需求就变得迫切，因为性能意味着时间、能源和金钱。

本文主要讨论现代商用硬件上高性能架构的若干设计原则，其中重点在于NUMA和hardware locality——拥有更深memory hierarchy的NUMA物理机与SMP上的高性能编程截然不同，需要充分利用hardware locality来释放它们的潜力。

## 商用硬件的现实
时代变了，服务端SMP已是过去时，曾经的众核异构未来已成为商用硬件的现实。

1. "Memory wall"成为性能的厚障壁: CPU和内存性能的差异实际上是在指数级增长。如今NUMA机器上的DRAM实际上可以看成当年的磁盘，所以以前的ordered map用红黑树(C++ std::map)，现代的ordered map用B树(Rust BTreeMap)以求cache性能。
2. "CPUs are networked devices"：把如今的高性能物理机CPU拆开来看，其实就是一个计算节点组成的网络，目前司内的icelake机器有2个numa sockets，128个虚拟核。核间通信、NUMA远端内存访问均是代价高昂的。既然网络算法需要考虑减少不必要的通信和数据拷贝，那么现代的高性能计算算法也需要。
3. todo

## 高性能架构的Heuristics
HPC三板斧分别是并行性(parallelism)，数据局部性(data locality)，特化(specialization)：

1. 并行是无处不在的，cpu层面上、物理机层面上、机房层面上均可横向扩展，因此充分利用硬件的并行性是提升系统整体性能的有效手段。“堆硬件”、“堆机器”是我们快速发展阶段(move fast and break nothing)，简单、粗暴、安全、稳定且烧钱的做法。并行性不仅可以简单利用，也可以更精细化地finetune：优化并行算法，提升内存分配的多核可扩展性，避免由不必要的context switch, data copy, cache false sharing，则可以减少烧钱。

2. 数据局部性。todo

3. 特化分为硬件特化和软件特化：用专业的硬件做专业的事，用直接的软件解决方案解决直接需求，毫无疑问可以提升效率，是高性能架构设计的必然选择，值得考虑的仅仅是技术风险。

HPC从学习角度看，一般有CPU、GPU两个分支，后文暂且搁置GPU tuning、CUDA、Hybrid model、模型压缩等话题，只讨论CPU相关的NUMA、OpenMP、Cache、SIMD、μArch。

我所在的互联网AI技术部门，GPU显然是目前更热门的话题。但如果把视角拉高俯瞰所有互联网计算系统，不难发现目前互联网场景中大部分计算密集应用实际上仍有着相当高的访存密度和访存不规则性(memory reference intensity & irregularity)：

1. 很多CPU-bound服务纯计算部分耗时不足一半，严格地更适合用CPU商用硬件，比如MIR、OLAP、全文检索。
2. 一些真正计算密集的领域，往往在业务上只有近实时约束，相对来说延迟不敏感，同样在实践中大量应用CPU而非GPU以硬件节省成本，如中小模型的推理、多媒体转码、特征提取。
3. 甚至某些GPU主导的算子优化领域，在CPU tuning做得足够深入之后，也能用更低总价达到近似的效果，比如稀疏科列斯基分解(sparse Cholesky factorization)，快速多极子算法(FMM)。

## 利用硬件拓扑局部性最大化众核效率
todo

## 被忽视的质量指标：透明度 
