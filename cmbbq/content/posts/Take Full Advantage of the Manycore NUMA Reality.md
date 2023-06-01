+++
title = "Take Full Advantage of the Manycore NUMA Reality"
date = "2023-06-13"
tags = ["sys"]
description = "当一个计算密集服务成为互联网规模的基础设施时，高性能计算的需求就变得迫切，因为性能意味着时间、能源和金钱。本文主要讨论现代商用硬件上如何通过share-nothing架构最大化众核+深内存层次结构时代的访存密集型CPU-bound应用的性能。"
showFullContent = false
+++

[WIP] [WIP] [WIP]

## NUMA和众核：曾经的超算概念，如今商用硬件的现实
时代变了，服务端SMP已是过去时，曾经的众核未来已成为商用硬件的现实。

1. "Memory wall"成为性能的厚障壁: CPU和内存性能的差异实际上是在指数级增长。如今NUMA机器上的DRAM实际上可以看成当年的磁盘，所以以前的ordered map用红黑树(C++ std::map)，现代的ordered map用B树(Rust BTreeMap)以求cache性能。
2. "CPUs are networked devices"：把如今的高性能物理机CPU拆开来看，其实就是一个计算节点组成的网络，目前司内的icelake机器有2个numa sockets，128个虚拟核。核间通信、NUMA远端内存访问均是代价高昂的。既然网络算法需要考虑减少不必要的通信和数据拷贝，那么现代的高性能计算算法也需要。
3. todo

## 网络IO密集应用语境下的share-nothing架构

## 访存密集应用语境下的share-nothing架构

## 虚拟化语境下hypervisor和vm如何利用hardware-locality最大化众核效率



