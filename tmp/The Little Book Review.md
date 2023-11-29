+++
title = "The Little Book Review"
date = "2023-11-27"
tags = ["sys"]
description = "正如ddia可被视为分布式系统方向的入门教程，lbdl是理想的深度学习101。"
showFullContent = false
+++

"The Little Book of Deep Learning"([lbdl](https://fleuret.org/francois/lbdl.html))是日内瓦大学CS教授François Fleuret写的一本适配手机屏的书，精简扼要地面向stem背景读者介绍深度学习。正如ddia可被视为分布式系统方向的入门教程，lbdl是理想的深度学习101。

精简，或者说压缩，正是深度模型的strength，也是这个信息过载时代的virtue。用A4纸打印这个小册子，读起来非常舒适。

<p float="left">
  <img src="https://cmbbq.github.io/img/tlb.jpg" width="32%" />
  <img src="https://cmbbq.github.io/img/tlb1.jpg" width="32%" /> 
  <img src="https://cmbbq.github.io/img/tlb2.jpg" width="32%" />
</p>

---

接下来是知识内化和梳理。

## 概述 

高维信号难以用规则系统分析，而深度网络则克服了这个困难，用具有大量权重的深层映射拟合出一个足够好（loss足够低）的近似函数——这个函数可以是高维信号到连续向量（回归）或离散值（分类）的映射，也可以是一种概率密度函数，总之，它能从数据分布中学习到某种紧凑且有区分能力的表征。

若数据样本不足，即使训练数据上表现良好，也可能在真实应用中效果不佳，这就是过拟合。
若模型能力不足，无法适应多变场景、准确捕捉输入输出的关系，训练时loss就高，则是欠拟合。

机器学习模型可以粗粒度地分为3类：
1. 回归模型：有监督，训练数据是输入信号和ground-truth数值的pairs，将高维信号映射到某个向量。
2. 分类模型：有监督，训练数据是输入信号和标签的pairs，将高维信号映射到有限标签集上。
3. 概率密度函数模型：无监督，训练数据就是输入信号本身。

## 训练 
### 损失函数
所谓训练，就是降低训练集上预测函数的损失函数（loss）的过程。

损失函数如何定义？对连续数值来说，均方差是一个标准选择。对概率密度来说，则用似然值——可令loss=-Sum(f(x;w))，其中f(x;w)是各个训练样本的标准化log概率。对分类任务来说，一般用交叉熵。

何为交叉熵？分类模型为N个类输出N个logits（其实LLM也是这样，为vocabulary里每个token生成对应的logits，表示未标准化的log概率），logits经过softmax，得到后验概率P(Y=y|X=x)，这是裸概率，各个类的概率加起来和为1。令loss=1/N Sum(-logP(Y=y_n|X=x_n))，这个loss即为交叉熵。交叉熵最小化，则真类别的概率最大化。

在度量学习中，虽然预测的值是连续的，但实际监督形式是分级，因为度量学习的目标是学习出样本之间可比较的距离，比如A、B、C三个点，其中A和B是同一个人脸的不同侧面，C是另一个人，或者A和B是同一首歌的翻唱，C是另一首歌，那就要求AB之间距离小于AC。因此度量学习一般采用contrastive loss或triplet loss。

损失函数通常只是一个代理指标，而非实际性能指标，以分类任务为例，显然直接性能指标应该是分类错误率，只不过这个指标的梯度没有携带有效指导信息——错误率函数和模型权重是完全剥离的，知晓错误率的变化不能在训练中帮助模型减小错误率。

损失函数还可以被设计为依赖于模型权重，从而对模型权重进行某种约束和控制。比如权重衰减（weight decay），一种防止过拟合的正则化技术，给损失函数里增加了一项模型权重的平方和，从而惩罚大的权重数值，偏好小的数值，进而减少训练数据对模型权重取值范围的影响。这么做会使训练集上性能下降，但有利于在未见过的数据集上更好地泛化。

### 自回归模型
自回归模型是NLP/CV等领域处理离散序列的关键方法。原理是利用条件概率的链式法则: 
```
  𝑃(𝐴∩𝐵)=𝑃(𝐴)𝑃(𝐵|𝐴),
  𝑃(𝐴∩𝐵∩𝐶)=𝑃(𝐴)𝑃(𝐵|𝐴)𝑃(𝐶|𝐴∩𝐵)
```
自回归模型输入是已有的T个token（每个token取值范围是大小为K的vocabulary集合），输出是K个候选token的logits。

token词汇域有限的场景是可计算的，条件概率的链式分解又使计算量降低——采样下一个token时，可以利用上一个token的概率，最终能生成符合联合概率分布的token序列。

训练自回归模型可以遍历各个步骤，把每一个逻辑时间节点上模型预测和真正的下一个token的交叉熵快照加起来，形成交叉熵loss。减小这个loss，即增大每个逻辑时间节点上模型预测token的似然。实际上监控的往往不是交叉熵，而是交叉熵(H)的指数，即困惑度perplexity(PPL)，PPL = 2^H。相比交叉熵，困惑度是归一化的，并不依赖输入序列的长度。

 