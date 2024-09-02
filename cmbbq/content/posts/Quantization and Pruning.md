+++
title = "Quantization and Pruning"
date = "2024-09-02"
tags = ["sys", "ai"]
description = "总结推理优化问题中相当重要的模型压缩技术——量化和剪枝。"
showFullContent = false
+++

模型压缩技术中，最实用的是量化，其次还可以尝试剪枝。量化降低精度，剪枝裁剪参数。

# Quantization
可以将k-bit量化问题视作将取值范围$(x_{min},x_{max})$的float数值$x$经过量化函数$g(x)$映射到取值范围$(0,2^{k-1})$的int型数值$Q$，并尽可能减少整体模型精度损失（well，如果没有端到端的模型accuracy/perplexity指标，也可以把目标设置为最小化output MSE/RMSE）的问题。$Q=round(g(x))$，

## Uniform vs Non-uniform
根据Q的分布是否为均匀分布，可讲量化器分为uniform quantizers和non-uniform quantizers[^8]。

Non-uniform quantization往往用几个离散的等级模拟其他分布（$x$真实的分布可能是lognorm分布、norm分布），以期在更稠密的取值范围内提升精度，缺点是计算量更大一些。

## Affine vs Scale
在uniform quantization中，变换函数又有两个选择：affine和scale。前者用一个仿射函数（$g(x)=kx+b$），后者只用（$g(x)=kx$），0在映射后仍为0，$Q$和$x$在0两侧对称，因此又叫对称量化，实际上是仿射量化的一个特例，由于去掉了offset，计算更简化了，也容易向量化。

## PTQ vs QAT
根据是否涉及backprop，量化可以分为PTQ(post-training Quantization)和QAT(Quantization Aware Training)这两类。QAT因训练成本高昂，难以扩展到大模型。因此大模型量化更多地使用PTQ。

## Dynamic vs Static 
静态精度量化把权重、激活函数、梯度统一转成低精度表示，比如W8A8量化。推理阶段W8A8完全是int8算术计算，不需要执行任何量化、反量化函数。

静态量化的参数（scale factor $k$、zero point $b$）是固定的。那么如何决定合适的$k$或$b$呢？静态量化往往需要通过在校准集上收集激活分布，寻找最小化MSE的最优解。但这种校准过程也会引入过拟合校准集的问题。在输入数据的分布非常明确，可以被校准集正确刻画的场景下，可以使用静态量化。

动态精度量化，又称weight-only量化，只把权重量化到低精度，激活仍保留高精度（因此模型变成了混合精度模型）。动态量化中，量化参数是推理时即时演算的，因此无需专门的校准阶段。推理时激活精度会动态调整精度（上限是模型中存储的激活精度，下限是权重精度，需施加量化函数到激活，或反量化函数到权重），因此保留了部分浮点数计算。

通常静态量化适合CNN，动态量化适合RNN、transformers。

量化器的计算量在混合精度模型（比如int4权重量化+fp16激活函数量化的gptq，或torchao QAT的8da4w，或llama.cpp的q4_k）中会影响推理性能。以llama.cpp的q4_k为例，中间张量（比如两个4-bits的sum，再用4-bits有几率产生overflow）有些需要用8-bits而非4-bits量化，可以把模型张量用计算量更高误差更少的量化器（比如non-uniform量化器，计算量虽大但不影响运行时开销），中间张量用计算量小的uniform量化器[^10]。

## LLM Quantization
- GPTQ[^17]：一种适用于大模型的oneshot量化方法，把所有权重都批量放入矩阵中，逐层量化，每次都最小化输出MSE。GPTQ采用int4/fp16混合精度量化，4-bit用于权重量化，激活函数仍用fp16。GPTQ利用二阶信息做误差补偿，但可能在重建过程中过拟合校准集，导致模型损失泛用性。

- AWQ[^18]：一种低比特weight-only的量化方法，只对权重进行量化，将激活函数和梯度保留为全精度。AWQ认为只有0.1~1%的权重是salient的，应跳过这些salient权重。激活后的分布比权重本身更加salient，因此AWQ根据激活分布寻找需要被跳过的权重。

- GGUF：在llama.cpp的k-quant体系中，qN_0表示N-bits scale量化，qN_1表示N-bits affine量化，qN_k代表特殊的block-wise量化，把原模型权重分块，每个块有自己的根据最大值简单算出的scaling factor（这样显然并不最优，后来又做了改进版本，见[^10]），salient权重被量化到高精度，其他的则量化到低精度，是混合精度的。以q2_k quant为例，salient权重被量化到4-bit，而其他权重则是2bit。q4_0则分别把所有权重都量化到4-bit。

- SmoothQuant[^23]：与per-channel激活量化不同，SmoothQuant把magnitudes做了一个smooth操作，避免过于剧烈的inter-channel variation。SmoothQuant把原本非常uniform的权重分别变得稍有起伏，但实际上仍然易于计算。

![smooth](https://cmbbq.github.io/img/smooth_quant.png)

## k-bit Inference Scaling Laws
根据[^5]中的三万五千次k-bit推理实验，模型总体积不变的情况下，4-bit精度几乎永远是最优解。

# Pruning
对应PTQ，也存在PTP(Post-Training Pruning)，本文主要讨论PTP，即无需高昂重训练成本（可以通过LoRA恢复）的剪枝。

## 结构化稀疏
结构化稀疏在特定维度（chanel、conv kernel）上对卷积、矩阵乘做剪枝操作，改变其shape，生成更小的模型。

LLM-Pruner[^20]、Torch-Pruning[^26]是针对LLM的结构化稀疏方法。Isomorphic Pruning[^27]则是近期针对ViT和现代CNN的SOTA方法。

## 非结构化稀疏
非结构化稀疏以每一个参数为单元进行稀疏，不改变参数矩阵shape，只是令其中部分值为零。要求底层推理实现能有效地利用矩阵稀疏性进行加速。

SparseGPT[^24]在不显著牺牲perplexity的前提下，对175B级别的模型使用（one-shot，无需retraining），达到60%非结构化稀疏。

SparseGPT将剪枝问题规约到超大规模稀疏回归实例上，用一种新的近似稀疏回归求解器高效求解，能在单GPU上几个小时内跑完100B级模型的稀疏。

## 半结构化稀疏
N:M Pruning[^25]是一种半结构化稀疏方法。SparseGPT这样的非结构化稀疏技术可以通过修改、适配成2:4 sparsity在A100上得到加速。


[^1]: [Quantization-Aware Training for Large Language Models with PyTorch](https://pytorch.org/blog/quantization-aware-training/)
[^2]: Y Lin, et al. FQ-ViT: Post-Training Quantization for Fully Quantized Vision Transformer. (PTQ, addressing extreme non-uniform distribution in attention maps & serious inter-channel variation in LayerNorm inputs) [[pdf]](https://arxiv.org/pdf/2111.13824)
[^3]: M. Sun, et al. A Simple and Effective Pruning Approach for LLMs(Wanda). [[pdf]](https://arxiv.org/pdf/2306.11695)
[^4]: [Exploiting NVIDIA Ampere Structured Sparsity with cuSPARSELt](https://developer.nvidia.com/blog/exploiting-ampere-structured-sparsity-with-cusparselt/)
[^5]: T. Dettmers, L. Zettlemoyer. The case for 4-bit precision: k-bit Inference Scaling Laws. [[pdf]](https://arxiv.org/pdf/2212.09720)
[^6]: [torchao](https://github.com/pytorch/ao/)
[^7]: [Accelerating Neural Network Training with Semi-Structured (2:4) Sparsity](https://pytorch.org/blog/accelerating-neural-network-training/)
[^8]: Raghuraman Krishnamoorthi. Quantizing Deep Convolutional Networks for Efficient Inference: A whitepaper. [[pdf]](https://arxiv.org/pdf/1806.08342)
[^9]: [Lloyd-Max Quantization ](https://www.khoury.northeastern.edu/home/gsharp/csg142-fall-2006/Lloyd-Max-Quant.pdf)
[^10]: [llama.cpp issue: Investigate alternative approach for Q4 quantization](https://github.com/ggerganov/llama.cpp/issues/397)
[^11]: A. Gholami, et al. A Survey of Quantization Methods for Efficient Neural Network Inference. [[pdf]](https://arxiv.org/pdf/2103.13630)
[^12]: Y. Choukroun, et al. Low-bit Quantization of Neural Networks for Efficient Inference.(low-bit/4bit) [[pdf]](https://arxiv.org/pdf/1902.06822)
[^13]: B. Jacob, et al. Quantization and Training of Neural Networks for Efficient Integer-Arithmetic-Only Inference.(int8) [[pdf]](https://arxiv.org/pdf/1712.05877)
[^14]: T. Dettmers, et al. 8-BIT Optimizer via Block-wise Quantization. [[pdf]](https://arxiv.org/pdf/2110.02861)
[^15]: [llama.cpp issue: Need help to understand q4_0, q4_1, q4_2, q4_3 quantization](https://github.com/ggerganov/llama.cpp/discussions/1121)
[^16]: [A Guide to Quantization in LLMs](https://symbl.ai/developers/blog/a-guide-to-quantization-in-llms/)
[^17]: E. Frantar, et al. GPTQ: Accurate Post-Training Quantization for Generative Pre-trained Transformers. [[pdf]](https://arxiv.org/pdf/2210.17323)
[^18]: Ji Lin, et al. AWQ: Activation-aware Weight Quantization for LLM Compression and Acceleration. [[pdf]](https://arxiv.org/pdf/2306.00978)
[^19]: [Low-Rank Pruning of Llama2](https://mobiusml.github.io/low-rank-llama2/)
[^20]: [LLM-Pruner: On the Structural Pruning of Large Language Models](https://arxiv.org/pdf/2305.11627)
[^21]: [llama.cpp quantize the intermediate results to 8-bits instead of 4-bits to gain accuracy](https://github.com/ggerganov/llama.cpp/pull/951)
[^22]: [llama.cpp k-quants](https://github.com/ggerganov/llama.cpp/pull/1684)
[^23]: G. Xiao, et al. SmoothQuant: Accurate and Efficient Post-Training Quantization for Large Language Models. [[pdf]](https://arxiv.org/pdf/2211.10438)
[^24]: E. Frantar, D. Alistarh. SparseGPT: Massive Language Models Can Be Accurately Pruned in One-Shot. [[pdf]](https://arxiv.org/pdf/2301.00774)
[^25]: A. Zhou, et al. Learning N:M Fine-grained Structured Sparse Neural Networks From Scratch. [[pdf]](https://arxiv.org/pdf/2102.04010)
[^26]: G. Fang, et al. DepGraph: Towards Any Structural Pruning. [[pdf]](https://arxiv.org/pdf/2301.12900)
[^27]: G. Fang, et al. Isomorphic Pruning for Vision Models. [[pdf]](https://arxiv.org/pdf/2407.04616)