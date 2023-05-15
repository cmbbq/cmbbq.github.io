+++
title = "On ABI"
date = "2022-08-02"
tags = ["sys"]
description = "本文总结了介于ISA和语言标准这两个简约协议层之间隔离了大量复杂度的抽象层次——系统语言的ABI。"
showFullContent = false
+++

前注：ABI在本文中特指系统语言的ABI，这里系统语言，即system programming language，指的是C，C++，Rust这样应用于系统编程的的编译语言。有时候我们也会讨论常用库或基础库的ABI，比如gcc5就把std::basic_string和std::list的实现改了，自然也就影响了跨版本的ABI兼容性，这种层面的ABI兼容性虽然也是一个工程上的疑难杂症，但不在本文的讨论范围内，毕竟源码都不同了。本文讨论ABI兼容性的是同一份源码在不同ISA、操作系统、编译器上对应二进制产物的兼容性。

系统语言的ABI、应用层软件的API、微处理器架构的ISA都是描述各自抽象层次互操作能力的接口。

ABI的独特之处在于它本身就是繁重的机制实现，而非轻量的协议声明——实现和声明之间有概念上的分野，这已经体现在法律上：复制ABI是赤裸裸的抄袭，而复制API或ISA则属于fair use。你显然可以合法地为他人写的API/ISA出书做注，Google实现自己的Java时合法复制了Oracle的Java API，无需license。

ABI之所以复杂，就是因为它处于两个抽象层次之间。ABI中的很大一部分内容既要向上对语言标准负责，向下要对ISA负责，必须遵循两个方向的约束，将复杂度留给自身，从而成就语言标准和ISA这两个抽象契约的简洁、干净，和人类友好。

ABI中的重要组成部分之一calling convention受到ISA的约束，i386中用cdecl, stdcall, fastcall, vectorcall, thiscall, amd64中用systemv, msnative, vectorcall，arm32用aapcs，arm64用aapcs64。ABI中也有很大一部分是指令集无关的（ISA-agnostic），比如name mangling、class layout。这些机制往往足够底层，又与其他指令集相关的部分有强耦合，加上历史因素，往往也只有一小部分能被写入语言标准。

如果ISA差异足够大，维护不同的ABI无疑是必要的，强行用一套ABI兼容不仅不自然，也不高效。以早期的Windows为例，微软就为早先的i386（IA-32）和后来的Intel IA-64提供了两套ABI。再后来，AMD赢得64位战争，Intel也遵循了前向兼容IA-32的amd64，又名x86_64, x64，IA-64就式微了。

ABI以ISA提供的指令、寄存器、内存管理能力为构件（building block），详尽且精确地描述如何实现一个系统语言在特定硬件、操作系统、编译器下的执行模型，并允许分开编译的产物之间能互操作。例如C ABI需定义调用约定（calling convention）、基础数据类型的表示、聚合数据类型的内存布局；C++03 ABI需额外定义异常机制、RTTI信息存储、虚表布局和动态绑定机制、重载函数/运算符/模板实例化所需的名字重置（name mangling）；C++11 ABI需进一步定义lambda的实现、自动类型推导机制等新增语言机制。

无论是C，还是C++都没有在语言层面制定官方的ABI标准，毕竟ABI标准本身也限制了实现的自由。目前最主流的Itanium C++ ABI号称被许多操作系统采用，能适配多数微处理器架构，被大多数编译器实现，包括gcc和clang这两个主要玩家。但值得注意的是，被多方支持，不意味着多方支持的是完全相同的东西。同样打着Itanium ABI名号，clang在arm32 linux上编译的C++库能给amd64 windows上的程序使用吗？显然不行，因为最基础的calling convention，甚至基础数据类型表示都不同。Itanium C++ ABI即使实现大一统，积极意义也仅限于允许一些依赖CPU无关规则的危险技巧（比如修改vtable，这种操作我们现在只能称之为hack）在相当多的环境下通用罢了——只要C++语言标准不将ABI纳入讨论范围，对ABI做出假设的技巧永远是危险的，想要创造新的语言机制，就必须在C++标准层面推进某种ABI共识。

那么系统语言是否应该对某种基于特定ABI实现的语言特性进行标准化呢？从C++的发展历史来看，这种做法已经有了先例，而且是相当危险冒失的。将非零抽象开销的dynamic exception、rtti引入C++客观上导致了社区分裂，有相当一部分C++使用者至今依然选择-fno-exeptions或-fno-rtti。近期的提案[Zero-Overhead Deterministic Exceptions: Catching Values](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2021/p2232r0.html)给出了一个对exception ABI进行改动的零开销异常机制，是对历史错误的亡羊补牢。系统语言的语言标准应审慎地只对ISA无关且零抽象开销的ABI规则进行标准化，以便在此基础上创造新的语言机制或为应用层开发提供便利。



