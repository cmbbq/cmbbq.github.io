+++
title = "Federated Learning vs In-center Distributed Training"
date = "2023-06-01"
tags = ["sys"]
description = "本文介绍基于个人设备的联邦学习和数据中心语境下的分布式训练。"
showFullContent = false
+++

[WIP]

# 联邦学习
[联邦学习](https://arxiv.org/pdf/1602.05629.pdf)由McMahan于2016年提出，指的是许多移动设备在一个中央服务器的编排下协作训练模型，保持训练数据离散，避免对用户数据进行收集，仅将客户端模型更新上传中央服务器汇总成新的全局模型的机器学习模式。

联邦学习的naive实现如下：
1. 中央服务器选定一些clients，让这些client下载模型。
2. 每个client根据自己的数据计算更新。
3. 每个client将更新（即新的完整模型）上传到中央服务器。
4. 中央服务器用某种方式（比如取平均）聚合这些模型得到一个全局模型。

## 上传模型更新到中央服务器开销大的问题
大量文献对联邦学习进行了讨论。[Federated Learning: Strategies For Improving Communication Efficiency](https://arxiv.org/pdf/1610.05492.pdf)指出联邦学习的naive实现的第3步很容易出现通信瓶颈，这是首当其冲的问题，提出了降低上行链路通信成本的两种方法：结构化更新，缩略更新。

联邦学习的问题可以被形式化地表述为：学习模型中的参数。全连接层的参数可用一个实数矩阵（这只是为了简化问题，所以讨论单个矩阵）表示 W ∈ R^(d1×d2)，shape为(#input × #output)，d1和d2表示输出维度和输入维度。卷积层kernel是4d tensor(#input × width × height × #output) ，需reshape到 (#input × width × height) × #output 。

用W(t)表示当前回合(t)的模型，W(i,t)表示本地更新后的模型，所谓更新就是H(i,t) = W(i,t) - W(t)。中央服务器可聚合得到新模型：W(t+1) = W(t) + η(t)H(t)，其中H(t) = Sum(H(i,t))/N，η(t)是learning rate。

### 结构化更新
结构化更新是指直接从一个用少量变量参数化的受限空间里学习更新，而不是学习整个模型的更新。

所谓结构化更新，也就是impose structure on updates，论文里提出两种结构，一种是低秩矩阵，另一种是随机掩码。

在结构化更新之低秩矩阵方法中，本地模型的更新H(i,t)必须是一个rank < k的低秩矩阵，这里k是一个固定的数字。将H(i,t)表示为H(i,t)=A(i,t)B(i,t)，其中A(i,t)∈ R^(d1×k)，B(i,t)∈ R^(k×d2)，在后续计算中，随机生成一个A(i,t)视为常数，然后优化B(i,t)。这样A的数据就不必上传，可以坍塌成一个随机种子，只需上传B(i,t)。这种优化，本质上是利用降维的压缩技术，使用矩阵将原始数据进行降维处理，然后使用重构矩阵将降维后的数据重新构建为原始数据。
A(i,t)每回合都为每个client随机生成一次。立刻就得到d1/k的上传开销减少。固定B，训练A，或者同时训练A和B都试过，效果不如固定A，训练B。对此的解释是A可视作重构矩阵（从变换后的向量中重新构建出原始向量），B是投影矩阵（将一个向量投影到另一个向量的子空间上）。固定A训练B实际上相当于解决这样一个问题：给定一个随机的重构矩阵，什么样的投影矩阵可以恢复最多的信息？

在结构化更新之随机掩码方法中，本地模型的更新H(i,t)必须是用某个预定义的随机掩码生成的稀疏矩阵。同样是每回合对每个client重新生成一次。稀疏掩码可以坍塌成一个随机种子，因此只需上传H(i,t)的非零值和种子。

### 缩略更新
缩略(sketched)更新，学习一个完整模型更新，学成之后，再通过有损的量化、随机旋转、子采样将其压缩后再发给服务器。

量化：将权重做概率量化，用更小的标量类型充当原始权重的unbiased estimator。

随机旋转：其实就是用一个随机正交矩阵乘一下，防止大部分数据都是0的情形，导致量化效果差。

子采样：原本上传的是H(i,t)，subsampling之后只上传一个随机子集。

## 算法、隐私、安全、工程等个层面的挑战
[Advances and Open Problems in Federated Learning](https://arxiv.org/pdf/1912.04977.pdf), 。

todo


## 通信延迟高的问题
还有一个关键问题——高通信延迟，在之前的论文中没被很好地address。[https://dga.hanlab.ai/assets/neurips21_dga.pdf](Delayed Gradient Averaging: Tolerate the
Communication Latency in Federated Learning)一文提出了一种延迟进行梯度平均的算法，用16节点是树莓派集群模拟现实世界中的移动节点和无线网络环境做了个实验，经验性地证明应用延迟梯度平均可以使联邦学习过程容忍高网络延迟，同时还不牺牲准确度。


todo

# In-center Distribtued Training

