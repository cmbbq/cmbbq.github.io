+++
title = "Diffusion Probabilistic Models"
date = "2024-06-04"
tags = ["ai"]
description = "本文是对朱军教授的分享——“用于生成高维数据的扩散模型”的笔记。值得注意的是DPM实践中巧妙使用了解析解，无论是前向过程的closed form $q(x_N|x_0)$，还是逆向过程中解析形式的方差估计，都大大提升了训练性能，体现了数学的精妙。"
showFullContent = false
+++

# 扩散模型生成高维数据
## 生成式模型范式
不同于判别式方法，生成式建模范式是：给定未知数据分布的一组IID[^1]数据$x_i \sim _{iid} p_D(x), i = 1,2,3,...$，去学一个参数空间为$\theta \in \Theta$的模型分布$p_{\theta}(x)$，令其逼近数据分布：$p_{\theta}(x) \approx p_D(x)$。

机器学习中，生成式模型的经典方法包括：
- Mixture of Gaussian for clustering
- Naive Bayes for classification
- Mixture of Experts(MoE) for unsupervised/supervised learning
- Probability graphical models, e.g. bayesian networks
- Nonparametric Bayesian methods
- Deep generative models

生成式模型天然具备构建基础模型的潜质，因为它的本质是对多元变量联合分布建模，只要能有效估计$p(x,y)$，自然就具备了对p(x)进行条件预测的能力——这也就构建出分类器，而且研究表明这样构建出来的分类器在半监督这种训练数据比较少的情况下表现出更高的数据利用率，此外也有些工作发现这种分类器在对抗干扰时表现得更鲁棒。

深度生成式模型的兴起，源自：
- 相比判别式模型，模型的表达能力变强了，能描述高维数据的复杂分布。
- 算法上有成熟的变分和马尔可夫链蒙特卡洛方法（MCMC）。
- 数据上更容易通过自监督或无监督方法利用大规模数据。
- 硬件上得到新GPU硬件对更大算力需求的支撑。

可微分神经网络深度生成模型用可微分DNN去学习随机变量之间的复杂关系，目标是将标准高斯白噪声变成一个自然场景下的真实分布（自然图片、声音、视频）。完全无监督下就能取得非常好的效果。这些模型根据概率密度函数定义可分为显式和隐式：
- 显式模型如VAE、Energy-based models、Auto-Regressive、Flow-based Models、DPM(Diffusion probabilistic models)直接描述预期产出数据的概率分布。
- 隐式模型如GAN、Moment-matching DGM则描述了一个变换过程，还需要通过一些准则去引导模型产生更符合预期数据分布的数据。

从训练目标来看，这些模型又可以分为最大似然估计（MLE）、Score-matching、对抗训练（Adversarial training）三类。
- Score-matching：Moment-matching DGM, Diffusion Models
- Adversarial training：GAN
- MLE：Everything else

## 扩散模型
物理过程中的扩散是随着时间推移，破坏结构，从有序到无序。

扩散模型中扩散过程也是逐渐给数据加高斯噪声，使其信噪比下降。

Song et al., ICLR 2021[^2]将一个扩散变换描述为：$q(x_i|x_{i-1}) = \mathcal{N}(x_i;\sqrt{1- \beta_i}x_{i-1},\beta_iI)$，令$\alpha_i = \prod_{k=1}^{i} 1 - \beta_i$，则有$q_{\alpha_N}(x_N|x_0) = \prod_{i=1}^{N} q(x_i|x_{i-1}) = \mathcal{N}(x_N;\sqrt{\alpha_N} x_0, (1-\alpha_N)I)$，冗长的递归表达式最终可以划归为一个简洁的closed form[^5]，因此可以很方便地定义N步前向过程的loss。其中$\beta_i$是一系列噪声乘数，可以是超参，也可以说reparameterization学习到的结果。对每个训练数据$x_0 \sim q_D(x)$，都可以构造一个离散马尔可夫链$\{x_0,x_1,...,x_N\}$，经过$N$次加噪，最终使之趋近高斯白噪。$q(x_N|x_0) \sim \mathcal{N}(O,I), N \rightarrow \infty$。

![SDE](https://cmbbq.github.io/img/sde.png)

上述过程的逆向去噪过程$p(x_{i-1}|x_i)$则是未知、需要学习或估算的——可以用变分近似的方法求解，比如用一个均值为$x_i$函数的高斯分布$\mathcal{N}(\mu_n(x_n), \theta_n^2I)$去近似$p(x_{i-1}|x_i)$，用KL散度最小化的方法使之逼近。

原理上来说Diffusion模型相对简单：
- 只有加噪去噪，不需要去学encoder和decoder，只需要根据加噪学去噪。
- 损失函数也比较简单。
- 数学上是严格保证收敛的。

## 大规模训练和高效数据生成
此前变分近似的做法中，噪声的方差参数一般是固定下来，不做优化的。清华大学TSAIL提出的Analytical-DPMs[^6]发现可以给出逆向过程中每个时间点的均值函数和方差的一个解析形式——这个形式和一些学者手工设计的一些形式也比较耦合——最终得到一个不需要任何额外训练的方差估计器。训练好的DPM，只需要插入一行代码，就能用上这个解析形式的方差估计。用这个估计值使每一步的方差估计变得更准，使所需的整体步数变少，折算下来有20~80倍的性能提升。后来这个方法也在Dall-E 2中使用了。

TSAIL团队的另一个工作DPM-Solver[^7]做了专门的求解器，使步数从几百步降低到十余步。

由于涉及加噪去噪，扩散模型的底层架构自然而然借鉴U-Net（CNN）。TSAIL团队的第三个工作，尝试把扩散模型和transformer结合，设计了U-ViT[^8]，在当时设置了5亿参数的（当时算最大的）大模型，证明对模型的可扩展性确实有帮助。同期有个工作DiT非常类似。Stable Diffusion 3.0就用的是DiT架构。

回顾前文所述的“生成式模型天然具备构建基础模型的潜质，因为它的本质是对多元变量联合分布建模，只要能有效估计$p(x,y)$，自然就具备了对p(x)进行条件预测的能力”，基于这种heuristics，TSAIL团队的另一个研究是UniDiffuser[^9]，目标是用一个模型解决原本marginal diffuser、conditional diffuser、joint diffuser这多个模型才能解决的多个任务。当时DALL-E 2和Stable Diffusion只能文到图，而UniDiffuser能图到文或文到图。

做完图像之后，又做了Vidu[^10]文生视频的工作，在时间轴上做了升维，实现了16s的生成。此外，还做了3D内容生成，CRM[^11]图生3D，ProlificDreamer[^12]文生3D，在空间上做了升维。在最新的工作Vidu4D[^13]中，做了4D（即sequential 3D）重建。

## 从生成到判别式分类器
生成式AI估计一个联合分布$P(x,y)$，基于贝叶斯定理可得$p(y|x) = \frac{p(x,y)}{p(x)} = \frac{p(y)p(x|y)}{p(x)}$，$y^* = \arg \underset{y\in \mathcal{Y}}{\max} p(y|x)$。

如果联合分布是准确的，那么这个分类器就是最优的，即所谓贝叶斯分类器。

此外，Chen et al 2024[^14]的工作表明可以将一个预训练好的生成式基座模型转化成一个对噪声鲁棒的分类器。

[^1]: IID stands for Independent and Identically Distributed
[^2]: Song et al. Score-based generative modeling through stohastic differential equations. ICLR 2021 [[arxiv]](https://arxiv.org/abs/2011.13456)
[^3]: Ho et al. Denoising diffusion probabilistic models(DDPM). NeurlPS 2020 [[arxiv]](https://arxiv.org/abs/2006.11239)
[^4]: In $\mathcal{N}(O,I)$, $I$ denotes the identity matrix, $O$ denotes the zero matrix.
[^5]: Some supplementary good ol' fashioned mathematical rigour: https://math.stackexchange.com/a/4568122
[^6]: Bao et al. Analytic-DPM: an Analytic Estimate of the Optimal Reverse Variance in Diffusion Probabilistic Models. ICLR 2022 [[arxiv]](https://arxiv.org/abs/2201.06503) 
[^7]: Lu et al. DPM-Solver: A Fast ODE Solver for Diffusion Probabilistic Model Sampling in Around 10 Steps [[arxiv]](https://arxiv.org/abs/2206.00927)
[^8] Bao et al. All are Worth Words: A ViT Backbone for Diffusion Models. CVPR 2023 [[arxiv]](https://arxiv.org/abs/2209.12152)
[^9] Bao et al. One Transformer Fits All Distributions in Multi-Modal Diffusion at Scale [[arxiv]](https://arxiv.org/abs/2303.06555)
[^10] Bao et al. Vidu: a Highly Consistent, Dynamic and Skilled Text-to-Video Generator with Diffusion Models [[arxiv]](https://arxiv.org/abs/2405.04233) 
[^11] Wang et al. CRM: Single Image to 3D Textured Mesh with Convolutional Reconstruction Model. NeurlPS 2023[[arxiv]](https://arxiv.org/abs/2403.05034)
[^12] Wang et al. ProlificDreamer: High-Fidelity and Diverse Text-to-3D Generation with Variational Score Distillation [[arxiv]](https://arxiv.org/abs/2305.16213)
[^13] Wang et al. Vidu4D: Single Generated Video to High-Fidelity 4D Reconstruction with Dynamic Gaussian Surfels [[arxiv]](https://arxiv.org/abs/2405.16822)
[^14] Chen et al. Robust Classification via Single Diffusion Model. ICML 2024. [[arxiv]](https://arxiv.org/abs/2305.15241)
[^15] Chen et al. Offline Reinforcement Learning via High-Fidelity Generative Behavior Modeling
 [[arxiv]](https://arxiv.org/abs/2209.14548)
[^16] Chen et al. Contrastive Energy Prediction for Exact Energy-Guided Diffusion Sampling in Offline Reinforcement Learning. ICML 2023. [[arxiv]](https://arxiv.org/abs/2304.12824)
[^17] Chen et al. Efficient Black-box Adversarial Attacks via Bayesian Optimization Guided by a Function Prior. [[arxiv]](https://arxiv.org/abs/2405.19098)
[^18] Hao et al. DPOT: Auto-Regressive Denoising Operator Transformer for Large-Scale PDE Pre-Training [[arxiv]](https://arxiv.org/abs/2403.03542)
[^19] Hu et al. Accelerating Transformer Pre-training with 2:4 Sparsity
