+++
title = "3rd-Gen Xeons"
date = "2023-12-29"
tags = ["sys"]
description = "介绍数据中心硬件之3代Xeon篇。"
showFullContent = false
+++

## SoC架构
目前数据中心应用的主力机型是Ice Lake、Cascade Lake机器，近未来计算/访存密集的场景则会用到Sapphire Rapids机器。

![cascadelake_icelake](https://cmbbq.github.io/img/cascadelake_icelake.png)

Ice Lake和Cascade Lake同属3代Xeon处理器的微架构，各方面变化不大，都是的monolithic mesh[^1]设计。

二者差异主要在制程（10nm vs 14nm）、最大核心数(常用机型逻辑核数 96 vs 128)、PCIe路数、内存通道数（单socket支持的DIMM[^2]数 16 vs 12）。

## 内存设备
内存通道数影响单机的内存上限，对内存数据库来说是一个重要指标，但对大多数检索应用来说一般用不着那么多，16个32GB DDR4提供512GB往往就够用。真正影响检索应用性能的是内存带宽。

通过```dmidecode --type 17```得到内存设备的状态。

可通过```Locator```和```Bank Locator```字段判断其所属numa node。

可检查是否所有内存模块有一致的速率，如果不同插槽内安装了不同速率的内存模块，即使配置了更高的速率，也会被最慢的内存模块拖累。

此外，```Memory Speed```字段反应了最大内存带宽，由内存控制器时钟频率和内存模块决定，但实际内存带宽可能会被故意配置得更低，需检查```Configured Memory Speed```字段。 有些内存插槽全部插满的超大内存机型为了减少能耗，提升稳定性，故意在BIOS中将带宽调低。


## PCIe设备
PCIe总线构成树形拓扑，Broadwell的ring架构时代有一个单一的IIO(integrated IO)组件作为Broadwell Ring的控制器，自然而然的设计是就是单PCIe root complex。Skylake的mesh架构时代每个die的边缘有多个PCIe$\times16$接口，单一IIO的功能被拆到了几个小的自有cache和流量控制器的独立IIO block中，自然可以支持多root设计。这就意味着GPU机器可以让某个PCIe$\times16$接口的下游GPU机器连入一个完全独立的PCIe mesh——这就给了拓扑配置上的灵活性，允许所有GPU只连单侧socket的CPU，避免CPU-to-GPU流量还要走跨socket的QPU/UPI通信。

可见Xeon Scalable中，Scalable的不仅仅是核心、DIMM，也包括PCIe root complex。

## PCIe设备
PCI号，如$\underbrace{\textcolor{grey}{0000}}_{domain}:\underbrace{\textcolor{red}{00}}_{bus}:\underbrace{\textcolor{cyan}{01}}_{slot}.\underbrace{\textcolor{yellow}{2}}_{func}$，由domain:bus:slot.func构成，前缀domain也可省略，因为一般都是0000。

3代Xeon的IOAT(Intel Input/Output Acceleration Technology) 设备即DMA设备，对应的驱动是ioatdma。

## DPDK思路
![eal_launch](https://cmbbq.github.io/img/eal_launch.svg)

1. 使用huge page：基于mmap在hugetlbfs中进行巨页物理内存申请。使用更大的内存页相比4KB默认页，所需的page table entry总数大大减少，可显著减小page table size和tlb size、降低tlb miss和page table walk开销，提升内存分配的连续性和内存访问的局部性，这些都有助于提升内存带宽。

2. 减少context switching开销：核心任务绑核，且尽可能减少阻塞IO和锁的使用。

3. 尽可能减少不必要的通信：尽可能避免核间通信和跨NUMA node的内存访问，通过DMA减少不必要的拷贝和缓冲。

3. Kernel Bypassing：用高效的用户空间实现替代内核网络栈。

## DPDK工具
1. 可用```usertools/dpdk-hugepages.py```在目标机器上查看、更新各numa节点上的hugepage设置。

2. 可用```usertools/dpdk-devbind.py --status```在目标机器上查看DMA和网络设备。

## Mellanox NIC
- 先进传输：XRC, DCT技术。
- Zero-Touch RoCE：无需任何特殊交换机配置，部署在聚合以太网[^3]上的RDMA。
    - ConnectX-4: 软件重传 
    - ConnectX-5: 硬件重传，支持tx-window
    - ConnectX-6 Dx: 采用新的拥塞控制协议ZTR-CC
- Nvidia PMDs[^4]: 
    - Nvidia提供了两个DPDK PMD，一个是ConnectX-3的mlx4(dpdk 2.0 release)，另一个是ConnectX-4~6的mlx5(dpdk 2.2 release)。
    - 支持裸金属、KVM、VMWare SR-IOV；支持x86_64、Arm和Power架构。

## DPDK & RDMA
DPDK通过VFIO在用户空间直接访问物理设备。DPDK和RDMA的数据链路同样是bypass kernel，控制链路却截然不同。
DPDK的大多卡数网驱动都是完全独立的纯用户空间驱动，通过MMIO(基于VFIO)和硬件设备通信。

[^1]: 其中monolithic是相对于chiplet/tile-based而言的单个huge die承载many-core的范式；mesh是相对于此前E5时代Broadwell Ring环状拓扑而言的网状拓扑。
[^2]: DIMM(dual in-line memory module)，即ram stick，内存条，DDR(Double Data Rate)技术的物理具现。
[^3]: 所谓聚合以太网（Converged Ethernet），就是一个承载多种类型流量（视频、音频、存储、控制）的单个以太网设施。
[^4]: 所谓PMD(Poll Mode Driver)即DPDK中bypass kernel，规避中断开销，直接与NIC硬件交互的设备驱动。实际上是用户态的PCIe驱动，基于MMIO实现和物理设备通信。
