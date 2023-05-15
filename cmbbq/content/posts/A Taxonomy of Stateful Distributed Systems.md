+++
title = "A Taxonomy of Stateful Distributed Systems"
date = "2021-07-08"
tags = ["sys"]
description = "本文讨论了CAP Theorem的局限性，梳理了基于一致性、可用性这两个理想属性间的权衡的更细致精确的有状态分布式系统分类学。"
showFullContent = false
+++


## CAP theorem的讨论范围是狭窄的
在分布式系统领域，CAP theorem广泛被引用，而且经常被用以指导超出它讨论边界的问题。

被形式化证明（见[Brewer's Conjecture and the Feasibility of Consistent, Available, Partition-​Tolerant Web Services](https://users.ece.cmu.edu/~adrian/731-sp04/readings/GL-cap.pdf)）的CAP theorem其实只局限在read-write storage场景：一个只包含get、set(x)两种操作的存储系统，这种系统被称为register。

在异步网络（指的是消息传递时间无界）中实现register是无法同时满足下列属性的：
1. Availability：所有发往register的请求最终都能完成。这和大多数真实系统的定义是不同的，因为真实系统不要求100%完成请求，只要保证SLA够高，同时又往往有一定时间约束，超时就会返回timeout错误。
2. Consistency：所有读、写操作均是linearizable的：B操作在A操作之后成功执行，则B看到的系统状态不能比A完成时的系统状态更旧。
3. Partition tolerance：允许网络丢包。

Partition tolerance默认是满足的，因此CAP-availability、CAP-consistency两种属性二选一。
离开了形式化证明的场景后，CAP theorem还有指导意义吗？答案是否定的——除非重新定义availability和consistency，将其推广到更通用的场景：从单对象单操作到多对象多操作的事务系统。
重新定义availability和consistency后的分布式系统分类学虽然和CAP theorem非常类似，但不能称之为CAP theorem。


## 一致性
更通用的“一致性”应定义为“并发系统中共享状态更新的可见性”。
现代微处理器、分布式系统、数据库的共性是——它们都是存在共享数据的并发系统。
当我们讨论一致性（consistency）时，可能指的是微处理器架构和系统编程领域的一致性模型，也可能指的是分布式系统领域的副本一致性，还可能指的是数据库领域的事务隔离性。这些领域的抽象层次不同，共同点是讨论的系统都是并发系统。

一致性模型（consistency model）是用于描述微处理器架构领域的多核并发场景下，各个处理器被允许的乱序程度——乱序的约束越少，效率越高，并发程序正确性越难保证。
1. 最强的strict consistency指的是任何写在任何时钟周期都立刻对任何处理器可见，显然不能推广到分布式系统领域。
2. 次之的sequential consistency指的是写操作顺序对于每个副本而言都是一致的，即各进程内部的program order一致，而不同进程执行的顺序可以不一致。这个概念最初也是Lamport在讨论multi-processor computer如何正确执行并发程序时提出的。和分布式系统的副本一致性无关。C++中std::memory_order_seq_cst即可保证线程内部的program order。
3. 更宽松的causal consistency指的是写操作中有依赖关系的那一部分的顺序是一致的，即各进程中的dependency order一致。现代CPU基本上都是out-of-order流水线，在保证dependency order这个底线后，能多乱序就多乱序。在C++中用std::memory_order_consume的load(A)和std::memory_order_acquire的store(B)配合，即可保证这个store之前所有写操作中load(A)依赖的那一部分对load(A)是可见的。如果每个依赖都保证Release-Consume ordering，则依赖链就有序，整体上可满足causal consistency。
4. 除了上述几个著名模型外，还有几十个不同方法、领域中应用的一致性模型，下图就包含了非事务分布式存储系统中种种一致性模型（详见[Consistency in Non-Transactional Distributed Storage Systems](https://arxiv.org/pdf/1512.00168.pdf)）。

![Consistency1](https://cmbbq.github.io/img/1.png)

并发程序显然可以很容易推广到分布式复制状态机，只是增加了网络延迟。因此，一致性模型可以推广应用到分布式系统中的副本一致性。以sequential consistency为例，增加了实时约束后就是分布式系统领域中更被广泛引用的linearizability，指的是单个被复制对象上的单个操作满足：A是写操作，B是对副本的读，A happened-before（因果律上的先于，见https://en.wikipedia.org/wiki/Happened-before） B，则A的写对B的读总是可见。比较一下C++的sequentially consistent ordering定义：everything that happened-before a store in one thread becomes a visible side effect in the thread that did a load。二者是一致的。

必须注意，迄今为止讨论的对象仅限单个被复制到不同副本中的对象上的单个操作，分布式存储不可能只存一个对象，有很多分布式存储支持事务或BatchWrite，涉及到多个对象上的多个操作。将单对象上单操作的可见性推广到多对象上多操作也不困难——事务的ACID隔离级别本质上就是将单个共享对象上单个操作的可见性推广到多个对象上一组操作上。下图的左侧就是数据库领域熟知的隔离级别，和分布式系统、微处理器架构、多核编程一样，乱序的约束越少，效率越高，并发程序正确性越难保证。

![Consistency2](https://cmbbq.github.io/img/2.png)


## 可用性
更通用的“可用性”应定义为“在施加某种约束后，系统仍最终能响应所有请求，无论网络分区持续多久”。
比CAP theorem更完善的分布式系统分类学可参考[Highly Available Transactions: Virtues and Limitations](https://arxiv.org/pdf/1302.0309.pdf)。
这篇论文里给出了新的可用性定义：

1. High availability：每个用户向运行中的系统发的请求，最终都会得到回复，无论网络分区持续多久。这就是CAP-availability，或者说traditional availability的标准定义。
2. Sticky availability：每当用户事务在一个数据库状态（该状态反应了之前该用户所有操作）拷贝上执行，最终都会得到回复，无论网络分区持续多久。 这比CAP-availability要求更高。
  - 只追求high availability，用户可以访问系统中的任何一个replica，不同操作在不同replica上响应也没关系；但若追求sticky availability，用户则需要保证连续的若干操作总是在同一个replica上。比如Dynamo这种multi-writer的分布式存储，不能写一会儿A节点，再写一会儿B节点。
3. Transactional availability：分布式系统文献的一致性模型大多考虑的都是单对象上单个操作的场景，而数据库文献中关注的是事务：多个对象上多个操作合起来称为一个事务。显然CAP-availability定义也不适用于事务。
  - 事务的replica availability：事务能为它需要访问的各个对象联系到至少一个replica。这个要求是比CAP-availability低的。
  - 事务的liveliness：假设我们让每个事务都abort，就可以保证100%的及时响应，完美实现CAP-Availability，但又有什么意义呢？因此还需要保证尽可能让事务commit，而不是abort。
  - 因此，最终给出的transactional availability定义是：对事务中每个数据都保证replica availability，并且最终能够在N次retry内commit成功，或internal abort（由事务自己主动选择的abort，而非系统实现将其abort）。
  - 更进一步还可以给出sticky transactional availability的定义：如果系统能保证sticky availability，则能保证transactional availability。

根据这种定义，可以将现有的事务系统的consistency（隔离级别）与其availability进行比较，得出下图中的结果：在新的分类学中，availability要求越高，consistency要求越宽松是成立的。

![Availability](https://cmbbq.github.io/img/3.png)
