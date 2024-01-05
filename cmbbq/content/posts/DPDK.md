+++
title = "DPDK is All You Need"
date = "2024-01-05"
tags = ["sys"]
description = "对于访存密集的数据中心应用来说，DPDK提供了非常好的性能工程范式。"
showFullContent = false
+++

对于访存密集的数据中心应用来说，DPDK提供了非常好的性能工程范式。本文只是浅尝辄止，汇集其中一部分值得借鉴的思想。

## EAL：用户空间库
EAL(Envionmemt Abstraction Layer)是DPDK面向用户的用户空间库，提供了各种有用的工具，比如：运行时对CPU特性进行检测，更适合现代硬件的内存管理，绑核和指定任务在某个核上运行。

`rte_eal_init()`是初始化EAL的函数，有多平台的实现：Linux、FreeBSD、Windows。以它为例，可大致了解DPDK EAL做了哪些工作。

`rte_eal_init()`是一个冗长的初始化过程，包含下列步骤：检查CPU类型是否是DPDK支持的、设置日志等级、检测各个socket上的各个cpu、enable每个逻辑核、初始化插件（加载共享库，比如一些PMD drivers）、初始化[tracing机制](https://doc.dpdk.org/guides/prog_guide/trace_lib.html)、解析各个设备的配置选项、初始化全局配置（主核id、逻辑核数、numa nodes数、iova模式[^1]、内存拓扑配置）、初始化中断处理机制、初始化多进程通用channel、扫描所有总线上的设备、初始化malloc heap、注册多进程action callbacks（热插拔支持）、初始化巨页信息[^9]、初始化内存和memzone、初始化HPET/TSC计时器[^8]、检查本地socket上的内存、创建主线程和子线程的通信信道、创建工作线程、绑核、在工作线程上启动dummy function、初始化服务、嗅探所有总线上的设备和驱动、启动服务、开启telemetry（提供ethdev stats、ethdev port list、eal parameters等状态查询）。

## 切割需要权限的工作
DPDK程序跑在用户态的一个前提是有内核驱动帮忙处理一些硬件设备注册、中断映射的事情。Linux上有几个可用的内核驱动，比如``vfio-pci``、``igb_uio``、``uio_pci_generic``。它们是泛用的PCI内核驱动模块，对所有PCI设备均适用。``igb_uio``基于Linux UIO提供所有类型的中断支持，比较古老，也比较简单，不支持IOMMU，因此IOVA mode只能用PA mode。``uio_pci_generic``和``igb_uio``类似，不过不支持MSI和MSI-X中断。``vfio-pci``支持基于IOMMU做IOVA映射，兼容VA mode和PA mode，如果能用`vfio`，就用`vfio`。

大多数设备需先从Linux内核驱动上解绑，然后再绑定到DPDK的内核驱动上。需用户在运行DPDK程序前，用`usertools`目录下的``dpdk-devbind.py``脚本做好设备和内核模块的解绑和绑定——这种需要root权限的准备工作也从用户态库中剥离了。

## 利用巨页，毕竟4KB已是古老时代的残响
DPDK基于mmap在hugetlbfs中进行巨页物理内存申请。使用更大的内存页相比4KB默认页(在x86上，DPDK目前支持2MB或1GB的巨页[^7])，所需的page table entry总数大大减少，可显著减小page table size和tlb size、降低tlb miss和page table walk开销，提升内存分配的连续性和内存访问的局部性，这些都有助于提升内存带宽。

## 尊重NUMA Node拓扑
DPDK的每个操作都是NUMA-aware的，提供的API默认是NUMA node亲和的，这让用户很难写出远端内存访问的代码。

## 尊重内存的硬件拓扑
DPDK的内存分配做得非常精细，利用了内存硬件拓扑等内存配置。

内存配置中有两个影响应用层的重要概念：内存通道（memory channels）、内存列（memory ranks）。

内存通道是CPU和内存之间的通信通道，理论上来讲内存带宽和通道数成正比，单个通道位宽64bit，2个就是128bit。内存通道数往往等于单socket支持的DIMM[^3]数，毕竟足够多的内存还需足够多的通道才能保证和CPU的互连。

CPU和内存之间是64bit接口，但单个内存颗粒（DRAM chips）的位宽可能是4bit、16bit，需要多个内存颗粒并联形成一个64bit的内存列，连接到同一组chip select上，从而保证各内存颗粒可被同时访问。内存模组必须至少形成一个内存列，才能和CPU通信。内存标签上的2R×8就是指列数为2，颗粒位宽8bit，一共16个颗粒。

> Depending on memory configuration on x86 arch, objects addresses are spread between channels and ranks in RAM

x86架构下内存通道和内存列在内存地址上interleaving，即均匀分布且递增，因此RAM可视作由$n_{chan}\times n_{rank}$个block组成的，其DIMM架构如下图。
![2chan4rank](https://cmbbq.github.io/img/2chan4rank.svg)

如图所示，内存池最好不要让对象的起始地址反复命中同一个channel或同一个rank，而是要充分利用不同内存通道、不同内存列，避免通道、列之间的负载不均，提升访存带宽。

DPDK的`mempool`给对象大小加恰当的padding，令内存池中的下一个对象的起始地址分布在不同内存通道和列中。具体实现可参考下面的代码，其中64B[^4]是x86的`cache line size`，也恰恰是一个`block size`，或`memory bus width`、`channel width`。无论如何，内存池里的地址都要首先保证是64B的整数倍，能整齐地放入cache，做到cache-friendly——事实上也是block friendly、memory bus width friendly，然后才是保证下一个对象的`block id`不能被$n_{chan}\times n_{rank}$整除。

```C 
static unsigned int arch_mem_object_align(unsigned int obj_size)
{
	unsigned nchan = rte_memory_get_nchannel();
	unsigned nrank = rte_memory_get_nrank();
	unsigned new_obj_size = (obj_size + 63) / 64; 
	while (get_gcd(new_obj_size, nrank * nchan) != 1)
		new_obj_size++;
	return new_obj_size * 64; 
}
```

## IOVA和VA模式的连续性
硬件不知VA，用户空间不知PA，DPDK的作用之一是bridge物理地址（PA）和虚拟地址（VA），因此给出一种IOVA(IO Virtual Address)是自然而然的设计。

DPDK的IOVA模式有两种：PA mode和VA mode。如果用了PA mode，则分配给DPDK的所有IOVA地址都是物理地址，或者说，也是IO虚拟地址，只不过这个IO虚拟地址的内存布局与物理地址完全相同。PA mode的缺点是需要root权限以读取页表，且可能会继承物理内存的碎片性。因此DPDK引入了新的VA mode，一方面无需root权限，另一方面基于IOMMU[^2]做物理内存的重映射，保证IO虚拟地址的连续性，并使其编码布局和一般虚拟地址在格式上做到匹配，这样就允许大片连续IOVA内存的申请。无论是硬件视角下，还是用户空间视角下，VA mode下IOVA内存区都是连续的。

![iova](https://cmbbq.github.io/img/iova.png)

## 固定物理地址刚好宜用DMA
尊重NUMA拓扑、尊重内存布局、IOVA的VA模式、使用巨页这些特性叠加起来，天然就决定了DPDK设计中，用户态进程用到的所有虚拟地址的underlying物理地址都是固定不变的——也就是说，这些地址是可以用于DMA的。DPDK用户态程序可不必涉足IO事务，让硬件自主代劳，通过固定不变的物理地址上的DMA事务完成。

## 多进程范式
DPDK还特别为多进程做了支持，允许一个主进程管理所有DPDK资源，多个子进程共享资源访问权。DPDK的额外努力在于它保证了子进程视野中的地址和peer进程、主进程视野中的地址是完全一样的，也就是说连指针都能跨进程传递——这听起来就相当危险，但性能肯定比各种安全的通信协作机制强。此外，DPDK还支持跨进程的全局锁，使多进程编程更接近多线程编程。

[^1]: [Memory in DPDK Part 2: Deep Dive into IOVA](https://www.intel.com/content/www/us/en/developer/articles/technical/memory-in-dpdk-part-2-deep-dive-into-iova.html)
[^2]: IOMMU是连接在DMA-capable IO总线和主存之间，将设备的物理地址映射到虚拟地址空间的专用硬件。物理机通常都支持IOMMU，以Intel为例，IOMMU技术即Vt-d：Intel® Virtualization Technology for Directed I/O。
[^3]: DIMM(dual in-line memory module)，即ram stick，内存条，DDR(Double Data Rate)技术的物理具现。
[^4]: 无论是i686还是x86_64，cache size都是64B，不过有些场景下合理的cache padding size是128，因为prefetcher一次取两个cacheline。
[^5]: [Memory in DPDK Part 4: 18.11 and Beyond](https://www.intel.com/content/www/us/en/developer/articles/technical/memory-in-dpdk-part-4-1811-and-beyond.html)
[^6]: [Memory in DPDK Part 3: 17.11 and Earlier Releases](https://www.intel.com/content/www/us/en/developer/articles/technical/memory-in-dpdk-part-4-1811-and-beyond.html)
[^7]: [Memory in DPDK Part 1: General Concepts](https://www.intel.com/content/www/us/en/developer/articles/technical/memory-in-dpdk-part-1-general-concepts.html)
[^8]: EAL通过`mmap`从用户空间访问HPET内核时间计数，暴露高精度计时器接口给服务层。
[^9]: EAL用`mmap`分配巨页物理内存，并将这些物理内存再通过内存池API暴露给服务层。
