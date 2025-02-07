+++
title = "Optimizing AI Inference"
date = "2024-08-28"
tags = ["sys", "ai"]
description = "总结AI工程中的推理优化问题。"
showFullContent = false
+++

## 优化的几个抽象层次
应用侧的AI Engineering大致上可以分为MLOps和推理优化。推理优化又可以分为几个抽象层次：
1. 最高层是模型优化（详见[^13]）：量化、知识蒸馏、参数剪枝、通道剪枝。
2. 中间层是图优化和通用优化：operator fusion/reconstruction、loop interchanges、data layout rewrites。
3. 底层是硬件算子：最优化的底层算子必然是最特化的（tensor-specific + hardware-specific + framework-agnostic），因此需将中层IR绑定到硬件算子（即后端算子选择），这些硬件算子往往需要根据设备信息，充分利用vectorization、parallelism、locality，做显式cache管理，通过合理的指令重排隐藏memory latency, 利用SIMD/AMX/DSP/GPGPU架构做memory tiling和minibatch block gemm。
4. 最底层还有LLVM层面的low-level codegen优化，负责最终生成优化的机器码。

第一层的模型优化减少了模型本身的总计算量，与底层优化正交。第二、第三层的推理优化归根到底都是硬件使能，只不过二层对几乎所有硬件都有效，三层则根据设备信息细化。从实现方法上讲，二、三层优化又可分为基于AI编译器的优化和手工优化。

## 基于编译的优化
编译，或者说DSLs + Optimizing Compilers，是解决领域问题优化的一个通用解法，为不同硬件提供可移植的优化。比如十年前就有Halide为图像和张量的并行计算提供了一个DSL+编译器，将算法规格本身和优化细节解耦。甚至更底层的gcc/llvm本身也是将算法和优化解耦的例子，在machine code codegen层面做的优化。

### ML Compilers的优势和劣势
如今的ML compilers是这种基于编译的思路的延续[^2]，其优势在于：
- 可移植性：硬件一方面在不断迭代更新，另一方面也不断有新硬件、新架构涌现。端边侧的设备要繁杂的多。Datacenter场景还好，但也面临N卡禁运，国产替代的问题，国产GPGPU还没有形成明确的一两家独大的格局[^10]，因此适配新硬件是近未来必需解决的事情。
- 将算法与优化清晰解耦：工程上可以提效，也有助于降低因复杂度爆炸而注入缺陷、使得项目逐渐失控、无法维护的风险。

其劣势在于：
- 不完备：在处理大多数场景、常规问题时性能不错，但总会遇到一些预料之外的edge cases，fallback to the slow path，性能骤然劣化，很难通过tweak DSL code生成更优的代码。
- 不灵活：考虑到编译器codegen产物往往不那么human-readble，工程上针对edge cases、ad-hoc需求做灵活的手工优化就比较困难。
- 难以真正击败专家手动优化：正如至今不存在能在性能上击败C/C++的函数式语言编译器，ML compiler也只是给出足够好的解，通常不及专家充分优化的C/C++实现。

### ML Compilers的工作流程
ML编译器的工作流是高层抽象到底层抽象的``lowering``过程。ML编译器后端包含各种``passes``，所谓pass就是lowering规则。最终会根据设备信息，形成一种硬件特化、张量特化的硬件算子描述，这种描述在TVM里叫做``schedule``，在Triton里叫做``plan``。再进一步CodeGen阶段，是从ML编译器自己的语言翻译到language compiler后端，比如LLVM IR，然后交由LLVM编译成可执行的machine code。

MLIR中``dialects``可对passes进行分层或分类，一个典型的dialects分层(见[^1])自上而下如是：
```
OpGraph -> TSOWB(e.g. late hlo) -> CGASel -> HHO(e.g. Linalg) -> MHA(e.g. stripe/affine) -> HLTSIR(e.g. vector dialects) -> TSIR(e.g. llvm)
```

Triton的大致流程如下：
```
面向用户的Python/C++的kernel代码
--> [ML Compiler前端，有时候可能只是某种动转静工具，forward一次，然后转写]
设备无关的 High-level IR
--> [ML Compiler后端Passes，图优化+算子选择+内存优化]
硬件特化的 Low-level IR [Schedule/Plan]
--> [ML Compiler后端Passes，把自己内部的Schedule/Plan翻译到LLVM IR]
LLVM IR 
--> [LLVM's NVPTX back-end，进入Language Compilation层面]
PTX
--> [CUDA ptxas assembler]
CUBIN
```

Intel MLIR graph compiler的lowering pipeline如下：

Computation Graphs -> linalg [^4] -> layout propagation [^7] -> tiling [^3] -> fusion [^8] -> micro kernel [^9] -> vector [^10] -> bufferization [^12] -> memory planning -> LLVM IR -> 交由LLVM处理。

上述lowering pipeline又可分为tensor-land和memref-land两个大的区块，bufferization之前都是tensor-land。在tensor-land，所有tensor操作都默认不是in-place的，哪怕是明显可以in-place的relu。在memref-land才会考虑内存访问的进一步优化。

## 手工优化
很多时候，目标模型的架构是确定的，目标机器的架构也就固定几种，ML compilers的可移植性优势——自动算子绑定/算子选择的优势——就近乎不存在了。

典型的例子是llama.cpp，和支撑它的ggml，llama.cpp+ggml通过一个人类可读的最小化C/C++项目，实现了各种精度的量化、自动微分、AVX/AVX2优化、Metal优化、flashatt算子、Multi-GPU pipeline并行等各个抽象层次上最直接有效的优化机制，最终也达成相当好的效果，尤其是在本地设备上。

这种手动优化的系统优势在于系统是透明的，程序员可以读懂整个系统，精准定位到涉及某个问题的代码行，从而具备ML编译方案中不具备的灵活性，适合应对ad-hoc需求，适用于模型架构和硬件设备稳定的场景。

手动优化的劣势是一旦出现新架构、新设备，就需要重写代码。


[^1]: [Linalg Dialect Rationale: The Case For Compiler-Friendly Custom Operations](https://mlir.llvm.org/docs/Rationale/RationaleLinalgDialect/)
[^2]: TVM可以视为Halide在ML领域的
[^3]: 算子分块，或者说矩阵乘法分块层，属于scf dialect，即structured control flow，之前叫LoopOps。
[^4]: ``Linalg`` is a DSL(a high-level MLIR dialect) for expressing linear algebra operations in MLIR, designed to solve the High-level Hierarchical Optimization (HHO box) in MLIR and to interoperate nicely within a Mixture Of Expert Compilers environment (i.e. the CGSel box). 
[^5]: [MLIR — Lowering through LLVM](https://www.jeremykun.com/2023/11/01/mlir-lowering-through-llvm/)
[^6]: [A friendly introduction to machine learning compilers and optimizers](https://huyenchip.com/2021/09/07/a-friendly-introduction-to-machine-learning-compilers-and-optimizers.html)
[^7]: 把layout调整好，比如$M\times N$调成$32\times 32$分块。
[^8]: 算子融合，比如elementwise+reduce的op fusion。
[^9]: micro kernel一般是手写的，比如分块后的最小粒度的matmul，一般是64*64的，直接交给编译器是做不好的，要对不同硬件要用不同指令，不同顺序，不同寄存器。
[^10]: 国产AI芯片处于混战阶段：华为Atlas系列、壁仞BR100、瑞芯微rk NPU、百度昆仑芯XPU、比特大陆（bm-se/sc）、寒武纪MLU、海光DCU、燧原GCU等。
[^11]: 各种vector操作，具体又可分为GPU dialect，Arm-Neon dialect、x86vector dialect（AVX，AVX512）、第四代Xeon的AMX dialect等。
[^12]: Bufferization in MLIR is the process of converting ops with tensor semantics to ops with memref semantics. 这一阶段会尽可能尝试将一些tensor计算的内存占用in-place化，终极目标是用更少的内存，减少copy次数。
[^13]: [Quantization and Pruning](https://cmbbq.github.io/posts/quantization_and_pruning)
