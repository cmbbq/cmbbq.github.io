+++
title = "Federated Learning"
date = "2023-06-01"
tags = ["sys"]
description = "联邦学习(Federated Learning)是指许多移动设备在一个中央服务器的编排下协作训练模型，保持训练数据离散，避免对用户数据进行收集，仅将客户端模型更新上传中央服务器汇总成新的全局模型的机器学习模式。与in-center的分布式训练相比，有其独特的优势和挑战。"
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

## 应对上传模型开销大的问题
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

## 完全去中心化联邦学习在各个领域面临的挑战
[Advances and Open Problems in Federated Learning](https://arxiv.org/pdf/1912.04977.pdf)中讨论了完全去中心化联邦学习在算法、隐私、安全、工程等个层面的挑战:
1. 中央服务器可能成为瓶颈和单点故障风险点。由此产生p2p/完全去中心化的设计思路。
2. 完全去中心化的算法需应对client可用性和网络稳定性的局限。
3. 设计一个试图达到最快收敛速度的模型平均策略是困难的。
4. 去中心化的场景会导致算法易受恶意攻击、不可靠数据或标注的威胁。
5. client的通信带宽和电量不足，，将已有的压缩算法移植到移动端比较困难。
6. 隐私问题：如何防止一个client重建另一个client的隐私数据。
7. 如何实现的问题：区块链作为分布式账簿本质上是一个最终一致的复制状态机。不过以太坊这样的区块链上的数据是公开的，如果要适用于联邦学习，还需要改造。
8. cross-silo场景（多个组织或公司一起训一个模型，但数据不能直接共享，比如多个银行一起训一个fraud detection模型）对数据进行分区，并增加incentive机制。
9. 通信和压缩瓶颈。
10. 公平性：联邦学习引入了新的bias来源——设备型号、地理位置、活动模式、本地数据集大小等。
11. 安全性计算问题：如何应对恶意服务器？如何应对外部攻击？

### Non-IID数据分布问题
Non-IID(independent & identically distributed) data问题：样本的统计属性没有均匀分布，对于任何client-partitioned数据集来说都是常见的。

论文中给出了多种non-identical client分布（考虑对特征x，标签y进行有监督学习，(x,y)~Pi(x,y)即为client i的本地分布，P(x,y) = P(y|x)P(x) = P(x|y)P(y)）：
- Feature distribution skew(covariate shift)，即不同client的P(y|x)相同，但特征的边际分布P(x)不同。比如笔迹识别中不同用户写相同的字，但笔法、书写习惯还是不同。
- Label distribution skew(prior probability shift)，即不同client的P(x|y)相同，但标签的边际分布P(y)不同。比如澳洲的日常动物识别应用，标签里就会频繁出现袋鼠，其他地区则不会。
- Same label, different features(concept drift)，即不同client的P(y)相同，但条件分布P(x|y)不同，相同标签在不同client上对应到了不同的特征。比如豪宅，在香港和加州尺度是不同的。
- Same features, different label(concept shift)，即不同client的P(x)相同，但条件分布P(y|x)不同，相同特征被标注成立不同标签。比如有人把熊猫标注为宠物，也有人把熊猫标注为猛兽。
- Quantity skew or unbalancedness，不同的client的数据量差异大。

违反independence同样常见，因为client的分布很容易收到触发训练的约束条件的影响：比如很多训练是利用夜间睡眠时间跑的，那就导致往往一个经度的地区的clients更容易遇到一起。

应对Non-IID数据，一个可行的方法是用一个不涉及隐私数据的全局共享小数据集做数据增强。此外，还可以限制一个用户每天能做的贡献上限，避免量上面的不平衡。此外，某些场景还可以把Non-IID从bug转化为feature，就训一个本地客制化的专用模型出来，提供个性化服务，而不是最终产生一个全局模型。文章后面还介绍了Non-IID数据集上的优化算法和收敛速率。

### 应对隐私问题：Split Learning
Split Learning是模型执行路径层面的横切，同时适用于训练和推理：最简单场景是让每个client一直前向算到某个特定的cut layer停下，将cut layer的输出（smashed data）传给中央服务器或peer接着算，于是就完成了无需数据共享就实现的前向传播。类似地，梯度反向传播是从最后一层到cut layer停下，仅将cut layer的梯度回传给client。这样整个过程中其他节点都不会直接访问本地数据。

考虑到cut layer的权重本身也能一定程度上反映底层的数据现实，Split Learning是否能提供形式化的隐私承诺仍然是个开放问题。

## 应对通信延迟高的问题
还有一个关键问题——高通信延迟，由于无线和长距离传输的特性而无法回避，但在之前的论文中没被很好地address。[Delayed Gradient Averaging: Tolerate the
Communication Latency in Federated Learning](https://dga.hanlab.ai/assets/neurips21_dga.pdf)一文提出了一种延迟进行梯度平均的算法，用16节点是树莓派集群模拟现实世界中的移动节点和无线网络环境做了个实验，经验性地证明应用延迟梯度平均可以使联邦学习过程容忍高网络延迟，同时还不牺牲准确度。

In-center环境下，同一个机柜的延迟<1us，同机房则是ms级别。无线环境大概是20ms，跨洋连接则至少100ms。在解决带宽问题后，延迟就成为最大瓶颈。这篇论文提出的DGA(Delayed Gradient Aggregation)算法的核心思路是延迟梯度平均到未来的某个迭代，即模型更新时接收过时的平均梯度，从而允许通信和计算流水线化。论文将问题形式化为：最小化随机函数的和。

![DGA](https://cmbbq.github.io/img/DGA.png)

N表示client数量，fi表示client i的stochastic损失函数。随机变量ζi关联一个mini-batch样本。


![DGA2](https://cmbbq.github.io/img/DGA2.png)

算法的主要思路是允许averaging通信过程中做本地更新（averaging通信和本地更新并行，）。FedAvg中clients在每轮结束发送参数到彼此，等averaging结束再开启下一轮。DGA里把averaging barrier延迟到了后续迭代(iteration，指的是本地更新的迭代)。因此clients可以立刻开启下一轮(round，指的是最外层循环，即一轮更新)。第一轮下收到外部信息时迭代已经发生了D次，延迟了D个迭代后进行梯度修正。理想情况下不存在通信延迟，D=0时，DGA恢复成最初的FedAvg。

1. clients在第t轮彼此发更新。
2. clients在本地更新后继续用最新的本地参数继续本地更新。(averaging通信延迟 > 单次甚至若干次本地更新)
3. 当其他client的第t轮信息到达，则client已进行了D次额外本地更新。
4. 将本地t轮梯度替换为接受到的平均梯度。

在最宽泛的场景下（延迟极高），延迟梯度可能要几个轮次之后才能抵达。这就需要将延迟参数D表示为D = sK + r，其中s>=0, r <=K。DGA仍能保证不同client只在最近D个梯度上是不同的，t-D轮之前的梯度都是一样的。

