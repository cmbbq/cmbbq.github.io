
+++
title = "eBPF Tracing for Memory-Stalled Applications"
date = "2023-12-07"
tags = ["sys", "kernel"]
description = "介绍eBPF——前沿的Linux系统的可观测性技术，以及基于eBPF的off-CPU性能分析。"
showFullContent = false
+++

## 现代workloads的瓶颈
如今使用CPU而不将计算offload到GPU等加速器的数据中心应用，大多是访存密集应用，例如倒排检索、向量检索、模型推理、分析型数据库。

纯粹instruction-bound的workload反而罕见。即使是最典型的计算密集场景，充满了矩阵乘加的大模型训推，实际瓶颈也出现在内存IO、off-package互连上。

现代硬件的发展趋势是越来越深的内存hierarchy，相应地，现代workloads（训推、检索、数据分析）的瓶颈也逐渐向内存总线、Cache、MMU、CPU-to-CPU互连、inter-node互连转移。

如今内存慢得就像过去的磁盘。Rust的hashmap用B树而非红黑树实现。ScyllaDB和DragonFly通过利用numa机器局部性以及更好的cache优化分别击败Cassandra和Redis。至少，对不自研加速器硬件的互联网公司的研发团队——即数据中心应用的开发者来说，矩阵乘加offloading到GPU/ASIC/AMX即可，访存优化为代表的互连优化才是性能工程的主战场。


## CPU Profiling和CPU利用率的局限性
对于访存密集应用来说，很多人被CPU利用率高误导，以为计算是瓶颈，于是做CPU profiling[^1]和计算优化，往往效果不佳。假设只有10%线程时间在CPU上跑，而70%在等内存读写，那无论CPU profiling做得多好，也没办法找到真正的瓶颈。```perf```是典型的CPU profiling工具，```perf record -F 1000```是按照1000Hz采样，对各个函数上的时钟周期可以给出比较准确的估计，但这种采样在阻塞阶段不生效。 类似地，CPU火焰图也只是给出不阻塞时的采样数据，CPU火焰图的最上沿的函数就是所有on—CPU函数，其耗时之和是CPU time之和，不包括off-CPU time。

CPU利用率在目前仍然被广泛使用，但它已经过时。CPU利用率的真实含义是“非空闲率”，即CPU没有跑idle thread的比例，也就包括了内存IO阻塞、网络IO阻塞、spinlock盲等。这个概念过于古老，它出现时尚不存在memory wall，cpu并不显著快于主存，这显然已经不适用于现代硬件语境，容易给人计算单元是系统瓶颈的错觉。

## eBPF off-CPU分析：in-kernel简报 
怎么度量off-CPU time？最简单的办法是应用层tracing，记录重点代码各个函数和代码块的耗时。大多数情况下，这其实就是最好的方案，精度也不错。不过相比火焰图还不够帅，也不够全面，毕竟在哪加时间戳依赖主观判断，有时候真正的瓶颈会出现在意想不到的地方。

Linux 4.8+[^3]可使用eBPF做off-CPU分析。比如eBPF工具[bcc/cpudist](https://github.com/iovisor/bcc/blob/master/tools/cpudist.py)，[bcc/offcputime](https://github.com/iovisor/bcc/blob/master/tools/offcputime_example.txt)。offcputime生成的call stacks可以直接用[flamegraph.pl](https://github.com/brendangregg/FlameGraph)绘制off-CPU火焰图。bcc包含了大量工具，其具体使用方式可参照其[官方tutorial](https://github.com/iovisor/bcc/blob/master/docs/tutorial.md)。

eBPF tracing与传统的off-CPU tracer(比如```perf```)相比，最显著的优势是不必把所有内核事件往用户空间dump（调度事件是非常频繁的，```perf```往往生成巨量数据，注入的额外开销太大，不仅仅是CPU开销，还有磁盘IO开销），而是在内核就按照某种可编程的规则做了总结，把精简的信息输出出来。

此外，内核支持eBPF后，各种off-CPU分析需求都可以统一用eBPF实现，不再需要针对不同场景使用或制作不同的工具了——此前用```perf```做事件追踪，用storage tracing做存储IO追踪，用内核统计数据观测调度时延，不成体系，且性能良莠不一。

eBPF追踪off-CPU时长的思路是在context switch事件结束时记录一次stack（off—CPU期间stack是不变的，一次足矣），为当前context的off-CPU时长增加线程睡眠时间。其伪代码如下：
```python
on context switch finish:
	sleeptime[prev_thread_id] = timestamp
	if !sleeptime[thread_id]
		return
	delta = timestamp - sleeptime[thread_id]
	totaltime[pid, execname, user stack, kernel stack] += delta
	sleeptime[thread_id] = 0

on tracer exit:
	for each key in totaltime:
		print key
		print totaltime[key]
```

## 所以，什么是eBPF？
简单说，eBPF是一个允许跑自定义代码做一些tracing和系统监控的in-kernel runtime，是BPF的升级版。

![ebpf](https://cmbbq.github.io/img/ebpf.png)

最初的BPF，即Berkeley Packet Filter，是一个用于报文过滤的几乎被遗忘的古老内核特性。eBPF在BPF基础上做了扩展，允许事件源从报文扩展到多种多样的事件源，eBPF VM有更大的存储空间，更多寄存器和64位word size——BPF事实上提供了一个in-kernel的沙盒环境，或者说虚拟机，安全且受限地执行用户定义的程序。因此eBPF机制的出现实际上在内核态程序、用户态程序之外创造了新的软件品类。

eBPF程序不是预编译或解释的，而是JIT CO-RE[^2]的，程序出错既不abort，也不panic，而是返回error message。内核态直接访问资源，用户态通过系统调用或fault访问资源，eBPF则是通过一些受限的helper访问资源——目前来说主要作用还是tracing，做一些可观测性上的工作。

![ebpf2](https://cmbbq.github.io/img/ebpf2.png)

eBPF把JIT编译器和安全验证器直接放到了内核里，用户态的bpf程序先经过parser变成AST，再做一些构造和语法分析，然后生成IR，最终生成优化后的bytecode。BPF bytecode作为输入进入内核的JIT和verifier再编译成机器码给CPU执行。

![ebpf3](https://cmbbq.github.io/img/ebpf3.png)

## eBPF的其他应用
eBPF除了用在可观测性上，还可以应用于网络，在L3/L4/L7做traffic control, monitoring或load balancing，比如libbpf ```tc```/```qdiscs``` library, ```XDP```(裸金属高性能可编程网络)/```Cilium```(高性能云原生网络)/```Katran```(传输层负载均衡)。

此外，eBPF还可以应用于安全领域。毕竟eBPF可以观测系统中的各种事件，比如监控某些敏感文件（/etc/passwd这种）是否被篡改。基于这种观测能力在加上一些安全相关的先验知识，就可以做一些安全工具。K8s的```seccomp```工具就是基于eBPF实现的。


[^1]: 区别于off-CPU分析，这里的CPU profiling指狭义的on-CPU分析，不考虑阻塞中的thread time。
[^2]: CO-RE: Compile Once Run Everywhere，也就是说BPF bytecode是可以relocate的。
[^3]: 不过Linux 5.x才有完整的CO-RE和BTF支持，其中BTF(BPF Type Format)是为eBPF设计的内核数据结构描述机制。