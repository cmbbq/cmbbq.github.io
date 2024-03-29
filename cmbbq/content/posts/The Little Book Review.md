+++
title = "The Little Book Review & Internalization"
date = "2023-12-15"
tags = ["ai"]
description = "正如DDIA可被视为分布式系统方向的入门教程，LBDL是理想的深度学习101。"
showFullContent = false
+++


"The Little Book of Deep Learning"([```LBDL```](https://fleuret.org/francois/lbdl.html))是François Fleuret写的一本适配手机屏的书，精简扼要地面向stem背景读者介绍深度学习。正如```DDIA```可被视为分布式系统方向的入门教程，```LBDL```是理想的深度学习101。

![tlb](https://cmbbq.github.io/img/tlb.jpg)

精简，或者说压缩，正是深度模型的strength，也是这个信息过载时代的virtue。用A4纸打印这个小册子，读起来非常舒适。

---

接下来是知识内化和梳理。

## 【一】概述 

高维信号难以用规则系统分析，而深度网络则克服了这个困难，用具有大量权重的深层映射拟合出一个足够好（loss足够低）的近似函数——这个函数可以是高维信号到连续向量（回归）或离散值（分类）的映射，也可以是一种概率密度函数，总之，它能从数据分布中学习到某种紧凑且有区分能力的表征。

若数据样本不足，即使训练数据上表现良好，也可能在真实应用中效果不佳，这就是过拟合。
若模型能力不足，无法适应多变场景、准确捕捉输入输出的关系，训练时loss就高，则是欠拟合。

机器学习模型可以粗粒度地分为3类：
1. 回归模型：有监督，训练数据是输入信号和ground-truth数值的pairs，将高维信号映射到某个向量。
2. 分类模型：有监督，训练数据是输入信号和标签的pairs，将高维信号映射到有限标签集上。
3. 概率密度函数模型：无监督，训练数据就是输入信号本身。

## 【二】训练 
### 损失函数
所谓训练，就是降低训练集上预测函数的损失函数（loss，记作$\mathscr{L}$）的过程。

损失函数如何定义？对连续数值来说，均方差是一个标准选择。对概率密度来说，则用似然值——可令$\mathscr{L}=-\sum f(x;w)$，其中$f(x;w)$是各个训练样本的标准化log概率。对分类任务来说，一般用交叉熵。

何为交叉熵？分类模型为N个类输出N个logits（其实LLM也是这样，为vocabulary里每个token生成对应的logits，表示未标准化的log概率），logits经过softmax，得到后验概率$P(Y=y|X=x)$，这是裸概率，各个类的概率加起来和为1。令$\mathscr{L}=-\frac{1}{N} \sum_{n=1}^N logP(Y=y_n|X=x_n)$，这个$\mathscr{L}$即为交叉熵。交叉熵最小化，则真类别的概率最大化。

在度量学习中，虽然预测的值是连续的，但实际监督形式是分级，因为度量学习的目标是学习出样本之间可比较的距离，比如A、B、C三个点，其中A和B是同一个人脸的不同侧面，C是另一个人，那就要求AB之间距离小于AC。因此度量学习一般采用contrastive loss或triplet loss。

损失函数通常只是一个代理指标，而非实际性能指标，以分类任务为例，显然直接性能指标应该是分类错误率，只不过这个指标的梯度没有携带有效指导信息——错误率函数和模型权重是完全剥离的，知晓错误率的变化不能在训练中帮助模型减小错误率。

损失函数还可以被设计为依赖于模型权重，从而对模型权重进行某种约束和控制。比如权重衰减（weight decay），一种防止过拟合的正则化技术，给损失函数里增加了一项模型权重的平方和，从而惩罚大的权重数值，偏好小的数值，进而减少训练数据对模型权重取值范围的影响。这么做会使训练集上性能下降，但有利于在未见过的数据集上更好地泛化。

### 自回归模型
自回归模型是NLP/CV等领域处理离散序列的关键方法。原理是利用条件概率的链式法则: 
$$P(A\cap B)=P(A) P(B|A)$$
$$P(A\cap B\cap C)=P(A) P(B|A) P(C|A\cap B)$$

自回归模型输入是已有的T个token（每个token取值范围是大小为K的vocabulary集合），输出是K个候选token的logits。

token词汇域有限的场景是可计算的，条件概率的链式分解又使计算量降低——采样下一个token时，可以利用上一个token的概率，最终能生成符合联合概率分布的token序列。

训练自回归模型可以遍历各个步骤，把每一个逻辑时间节点上模型预测和真正的下一个token的交叉熵快照加起来，形成交叉熵loss。减小这个loss，即增大每个逻辑时间节点上模型预测token的似然。实际上监控的往往不是交叉熵，而是交叉熵(H)的指数，即困惑度perplexity(PPL)，PPL = 2^H。相比交叉熵，困惑度是归一化的，并不依赖输入序列的长度。

训练时，每个时刻都需要重新计算之前已经算过的，考虑到总体逻辑时间/步骤数目往往相当长，成百上千，甚至上万，这样的计算显然非常低效。解决方案是设计一个一次性预测所有逻辑时间（T）上的logits向量的模型——$f: \{1,...,K\}^T \rightarrow \mathbb{R}^{T\times K}$，并确保t时刻的输入$x_t$对应的logits $l_t$只依赖于$x_1, x_2, x_3, ... x_{t-1}$。这种模型即因果模型（causal models），其原则是不让未来影响过去。
 
![causal](https://cmbbq.github.io/img/causal.png)

因果模型训练时可以用完整的序列计算output，一次性最大化序列中所有token的概率，最终也等价于最小化per-token交叉熵。

自然语言处理中有一个重要技术细节，即如何进行token的表示，可以是最低粒度的单符号，也可以说整个词。进行token表示的算法过程叫做tokenizer。一个标准做法是Byte Pair Encoding[Sennrich et al., 2015][^1]。

### 梯度下降
除了线性回归这种简单特例，一般最优权重$w^*$不会有closed-form expression。这种情况下最小化函数的工具是梯度下降：将权重初始化为随机的$w_0$，然后反复迭代，每次迭代都朝梯度方向修改权重使loss逐步降低，即每次迭代都令$w_{n+1} = w_n - \eta \nabla \mathscr{L}_{|w}(W_n). $ 其中$\eta$即学习率，如果设置得太小，训练可能太慢，而且容易卡在局部最小，如果设置得太大，则容易在最低点附近左右横跳。

![gd](https://cmbbq.github.io/img/gradient_descent.png)

对于每个点$w$来说，梯度$\nabla \mathscr{L}_{|w}(w)$就是能最大化$\mathscr{L}$增量的方向。因此梯度下降就可以通过每次迭代中减去学习率*梯度的值，使所有迭代串联起来可形成一个接近最优的最小化$\mathscr{L}$路线。

实践中，所有loss均能表示为多个小样本甚至单样本loss的均值：$\mathscr{L} = \frac{1}{N} \sum_{n=1}^N \ell_n(w) $，其中$\ell_n(w)=L(f(x_n;w),y_n)$，因而梯度可表示为下式：

$$\nabla ℒ_{|w}(w) = \frac{1}{N} \sum_{n=1}^N \nabla \ell_{n|w}(w)$$

全量计算梯度开销较高，可以用局部求和估计全量求和（要求做好数据shuffling，数据的stochasticity消除估计的偏倚）。为了让计算能放进内存，标准的做法是把完整训练集分成相当多（可以是百万级）batches，从每个batch得到一个梯度的估计，然后根据这个估计去更新权重，这种做法即mini-batch SGD(stochastic gradient descent)。这种算法有很多变种，比如Adam[Kingma and Ba, 2014][^2]。

### 反向传播
给定$\ell(w)=L(f(x;w),y)$，怎么计算$\nabla\ell_{|w}(w)$呢？考虑到$f$和$L$都是标准张量运算的组合，它们与任何数学表达式一样，基于链式法则可以得到其表达式。

![bp](https://cmbbq.github.io/img/bp.png)

简单起见，将一个深度为$D$的模型表示为$f = f^{(D)} \circ f^{(D-1)} \circ ... \circ f^{(1)}$。则前馈过程即按顺序计算$x^{(d-1)} \rightarrow x^{(d)}$，即$x^{(d)} = f^{(d)}(x^{(d-1)};w_d)$，直到最终得到$x^{(D)}$作为模型输出。

反向传播过程则是反过来计算$\nabla\ell_{{|x}^(d-1)} \leftarrow \nabla\ell_{{|x}^(d)}$以及$\nabla\ell_{|w_d} \leftarrow \nabla\ell_{{|x}^(d)}$，其中$\nabla\ell_{{|x}^(d-1)}$[^3]是$\nabla\ell_{{|x}^(d)}$[^4]和$J_{f^{(d)}|x}$[^5]乘积。而我们在训练过程中实际关心的另一个梯度$\nabla\ell_{|w_d}$是$\nabla\ell_{{|x}^(d)}$和$J_{f^{(d)}|w}$[^6]乘积。

深度学习训练框架主要处理的就是隐藏反向传播梯度计算的复杂度，即提供自动求导/计算求导能力，该技术在深度学习之前也有广泛应用，详见AutoGrad [Baydin et al., 2015][^7]。

显然反向传播的矩阵计算量两倍于前馈推理（每层多了一次权重的反向传播）。反向传播的内存需求也远大于前馈推理，因为每一层的$x^{(d)}$都要保留在内存中，而非推理则不需要保留，只要留着最新的。解决内存占用过高的技术包括：reversible layers[Gomez et al., 2017][^8]和checkpointing[Chen et al., 2016][^9]。

深度模型的一个问题是梯度消失（见[Glorot and Bengio, 2010][^11]），即经过很多次反向传播后，数值变得太大或太小。常规应对做法是gradient norm clipping[Pascanu et al., 2013][^10]。

### 自监督训练
GPT在大规模无标注训练集上训练就足以处理很多任务，比如翻译（见[Radford et al., 2019][^12]），这就是典型的自监督训练应用，其最重要的优势是可以利用超大规模的未标注数据，将训练数据规模的边界再向前推进。

## 【三】构件
### 线性层
全连接层是最基础的线性层构件，可用$D \times D'$的矩阵$W$和一个bias向量$b$表示。它实现了一个能泛化到任意张量形状的仿射变换，给定任意输入$X$，其形状为$D_1 \times \dots \times D_k \times D$，全连接层计算得到一个输出$Y$，其形状为$D_1 \times \dots \times D_k \times D'$：$\forall d_1,\dots,d_K, Y[d1,\dots,d_K] = WX[d1,\dots,d_K] + b $


全连接层处理高维数据时，参数量太大。此外全连接层假设输入输出之间存在复杂非线性关系，忽视了更简单的结构化规律[^13]。然而高维信号普遍有这种强结构，比如图片兼具short-term关联和抵抗变换、缩放、对称的统计学静态性。相较而言，卷积层能更好地捕捉信号的空间结构——因为卷积层权重被输入信号的不同部分共享，这些权重因而能学习到某种局部空间结构，比如图片的边缘、形状、角。多层卷积是高维信号（图片、声音）的常用降维工具。

一维卷积以$D \times T$张量$X$为输入，对每个$D\times K$子张量施加仿射变换$\phi(\cdot;w): \mathbb{R}^{D\times K} \rightarrow \mathbb{R}^{D'\times 1}$，将$D'\times 1$结果依次存入$Y$中。 

一维反卷积则是以$D \times T$张量$X$为输入，对每个$D\times 1$子张量施加仿射变换$\phi(\cdot;w): \mathbb{R}^{D\times 1} \rightarrow \mathbb{R}^{D'\times K}$，将结果加起来形成的$D'\times K$张量存入$Y$中。

下图中，$D=3, K=5, D'=4$。

![1dconv](https://cmbbq.github.io/img/1dconv.png)

![2dconv](https://cmbbq.github.io/img/2dconv.png)

一维卷积往往用于处理序列数据或时序数据。二维卷积则常用于处理图片，或其他以2D矩阵为输入的任务。转置卷积/反卷积则主要用于GAN、VAE这样的生成式模型中，从一个低维特征膨胀到一个高分辨率图片。

### 激活函数
如果模型中仅使用线性组件，则整体也是线性操作，因此必须要引入非线性性，这种非线性性通常由激活函数实现。最常用的激活函数是ReLU [Glorot et al., 2011][^14]。ReLU之前则是双曲正切函数Tanh。

还有一些激活函数思路和ReLU差不多，保证正数不变，压缩负值，如：Leaky ReLU[Maas et al., 2013][^15]，GELU [Hendrycks and Gimpel, 2016][^16]。

### 池化
池化是降维，减少信号大小的经典策略，将若干邻近值取max或avg合并出一个值。

### 随机失活
Dropout[Srivastava et al., 2014][^17]层无可训练参数，只有一个超参$p$，在训练时用于以概率$p$随机关闭一些neuron，避免个别neuron对整体的影响，迫使其他neuron用略微不同的权重代劳。因此dropout是训练时防止过拟合的正则化工具。推理时dropout是关闭的。

### 归一化层
归一化可用于抵抗梯度消失。最主要的归一化层是Batch Normalization[Ioffe and Szegedy, 2015][^18]，由超参$D$和可训练参数$\beta_1, \dots,\beta_D$和$\gamma_1, \dots,\gamma_D$组成。给定一批$D$维样本$x_1, \dots, x_B$，先计算每个维度的均值$m_d = \frac{1}{B} \sum_{b=1}^B x_{b,d}$ 和方差 $v_d  = \frac{1}{B} \sum_{b=1}^B (x_{b,d} - m_d)^2$。

然后再对每个b，计算均值为0方差为1的归一化值$z_{b,d} = \frac{x_{b,d} - m_d}{\sqrt{v_d + \epsilon}} $，再算出最终结果$y_{b,d} = \gamma_d z_{b,d} + \beta_d$，这个最终值的均值为$\beta_d$，方差为$\gamma_d$。

![norm](https://cmbbq.github.io/img/norm.png)

### 残差连接
跳跃连接（skip connections，见[Long et al., 2014][^19]; [Ronneberger et al., 2015][^20]）同样可对抗梯度消失。实际上跳跃连接不是一个layer，而是一种让某些层的输出跳过一些中间层嫁接到后面的设计。这种设计允许更原始的信号在更后面的层中得到“反思”。

跳跃连接的实用实现是残差连接（residual connections），直接把两种信号求和，而且跳跃的层数不算多。这种设计允许信号在穿越某些原本会梯度消失的层时得以幸存。基于残差连接，何凯明构建了ResNet[He et al., 2015][^21]，Google设计了Transformer[Vaswani et al., 2017][^22]。

### 注意力层
已有的组件缺乏将局部信息和张量中较远位置的信息结合起来的能力，Attention Layer则专长于此道——通过为所得张量的每个组件与输入张量的每个组件计算注意力得分，不受局部性约束地，在整个张量范围内对特征进行平均[^22]。

给定$N^Q\times D^{QK}$维queries张量$Q$，$N^{KV}\times D^{QK}$维keys张量$K$，$N^{KV}\times D^V$维values张量$V$，通过```Attention操作```$att(K,Q,V)$计算得到$N^Q\times D^V$维的张量Y：

$$Y = att(K,Q,V) = \underbrace{softargmax(\frac{QK^T}{\frac{1}{\sqrt{D^{QK}}}})}_A V$$

整个过程分两步，第一步先计算每个query index $q$和每个key index $k$的注意力得分，即queries和keys点乘后的```softargmax```结果：$A_{q,k} = \frac{exp(\frac{1}{\sqrt{D^{QK}}} Q_q \cdot K_k ) }{\sum_l exp(\frac{1}{\sqrt{D^{QK}}} Q_q \cdot K_l)}$，其中$\frac{1}{\sqrt{D^{QK}}}$是一个缩放参数，用于保证取值范围在$D^{QK}$变大时大体不变。

![att](https://cmbbq.github.io/img/attention.png)

得到注意力得分$A_{q,k}$后，再进行第二步计算：$Y_q = \sum_k A_{q,k}V_k$。注意力得分即queries和keys之间的匹配程度，匹配程度越高，该权重就越高。如果某个query和某个key匹配度达到极限，注意力得分接近1，则直接拿到这个key对应的value。如果和好几个key都有中等水准的匹配，则按注意力得分做加权平均。

## 其他话题
```LBDL```还讨论了各种深度学习模型架构和应用，如多层感知机、卷积网络、注意力模型、RNN、Autoencoder、GAN、图神经网络、GPT、Diffusion。


[^1]: R. Sennrich, B. Haddow, and A. Birch. Neural Machine Translation of Rare Words with Subword Units. CoRR, abs/1508.07909, 2015. [[pdf]](https://arxiv.org/pdf/1508.07909).
[^2]: D. Kingma and J. Ba. Adam: A Method for Stochastic Optimization. CoRR, abs/1412.6980, 2014. [[pdf]](https://arxiv.org/pdf/1412.6980).
[^3]: $\nabla\ell_{{|x}^(d-1)}$即$f^{d-1}$的变量$x^{d-1}$对应的损失函数梯度。
[^4]: $\nabla\ell_{{|x}^(d-1)}$即$f^{d}$的变量$x^{d}$对应的损失函数梯度。
[^5]: $J_{f^{(d)}|x}$即第d个layer函数$f^{(d)}$相对变量x的Jacobian，雅可比矩阵，即函数的一阶偏导数以一定方式排列成的矩阵。
[^6]: $J_{f^{(d)}|w}$即第d个layer函数$f^{(d)}$相对权重w的Jacobian。
[^7]: A. Baydin, B. Pearlmutter, A. Radul, and J. Siskind. Automatic differentiation in machine learning: a survey. CoRR, abs/1502.05767, 2015. [[pdf]](https://arxiv.org/pdf/1502.05767).
[^8]: A. Gomez, M. Ren, R. Urtasun, and R. Grosse. The Reversible Residual Network: Backpropagation Without Storing Activations. CoRR, abs/1707.04585, 2017. [[pdf]](https://arxiv.org/pdf/1707.04585).
[^9]: T. Chen, B. Xu, C. Zhang, and C. Guestrin. Training Deep Nets with Sublinear Memory Cost. CoRR, abs/1604.06174, 2016. [pdf](https://arxiv.org/pdf/1604.06174).
[^10]: R. Pascanu, T. Mikolov, and Y. Bengio. On the difficulty of training recurrent neural networks. In International Conference on Machine Learning (ICML), 2013. [pdf](https://proceedings.mlr.press/v28/pascanu13.pdf).
[^11]: X. Glorot and Y. Bengio. Understanding the difficulty of training deep feedforward neural networks. In International Conference on Artificial Intelligence and Statistics (AISTATS), 2010. [pdf](https://proceedings.mlr.press/v9/glorot10a/glorot10a.pdf).
[^12]: A. Radford, J. Wu, R. Child, et al. Language Models are Unsupervised Multitask Learners, 2019. [[pdf]](https://d4mucfpksywv.cloudfront.net/better-language-models/language_models_are_unsupervised_multitask_learners.pdf).
[^13]: 这就是所谓的全连接层的```inductive bias```。
[^14]: X. Glorot, A. Bordes, and Y. Bengio. Deep Sparse Rectifier Neural Networks. In International Conference on Artificial Intelligence and Statistics (AISTATS), 2011. [[pdf]](http://proceedings.mlr.press/v15/glorot11a/glorot11a.pdf).
[^15]: A. L. Maas, A. Y. Hannun, and A. Y. Ng. Rectifier nonlinearities improve neural network acoustic models. In proceedings of the ICML Workshop on Deep Learning for Audio, Speech and Language Processing, 2013. [[pdf]](https://ai.stanford.edu/~amaas/papers/relu_hybrid_icml2013_final.pdf).
[^16]: D. Hendrycks and K. Gimpel. Gaussian Error Linear Units (GELUs). CoRR, abs/1606.08415, 2016. [[pdf]](https://arxiv.org/pdf/1606.08415). 
[^17]: N. Srivastava, G. Hinton, A. Krizhevsky, et al. Dropout: A Simple Way to Prevent Neural Networks from Overfitting. Journal of Machine Learning Research (JMLR), 15:1929–1958, 2014. [[pdf]](https://jmlr.org/papers/volume15/srivastava14a/srivastava14a.pdf).
[^18]: S. Ioffe and C. Szegedy. Batch Normalization: Accelerating Deep Network Training by Reducing Internal Covariate Shift. In International Conference on Machine Learning (ICML), 2015. [[pdf]](http://static.googleusercontent.com/media/research.google.com/en//pubs/archive/43442.pdf). 
[^19]: J. Long, E. Shelhamer, and T. Darrell. Fully Convolutional Networks for Semantic Segmentation. CoRR, abs/1411.4038, 2014. [[pdf]](https://arxiv.org/pdf/1411.4038). 
[^20]: O. Ronneberger, P. Fischer, and T. Brox. U-Net: Convolutional Networks for Biomedical Image Segmentation. In Medical Image Computing and Computer-Assisted Intervention, 2015. [[pdf]](https://arxiv.org/pdf/1505.04597.pdf).
[^21]: K. He, X. Zhang, S. Ren, and J. Sun. Deep Residual Learning for Image Recognition. CoRR, abs/1512.03385, 2015. [[pdf]](https://arxiv.org/pdf/1512.03385).
[^22]: A. Vaswani, N. Shazeer, N. Parmar, et al. Attention Is All You Need. CoRR, abs/1706.03762, 2017. [[pdf]](https://arxiv.org/pdf/1706.03762)