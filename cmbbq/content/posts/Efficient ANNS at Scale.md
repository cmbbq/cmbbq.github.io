+++
title = "Efficient ANNS at Scale"
date = "2023-09-12"
tags = ["ai", "sys"]
description = "如何在十亿、百亿级特征库上做高效的向量检索"
showFullContent = false
+++

向量检索分为KNN和ANN，KNN就是用brute-force计算出query向量和所有索引向量的相似度，取前k个。ANN则是通过近似手段加速计算，以误差换效率。

如何做高效的向量检索？方法有二，一是提升编码率（不仅能降低内存占用，也能加速计算，因为便于SIMD且减少访存）的量化（IVF向量量化、PQ量化、ScaNN各向异性量化、4bit量化），二是缩小搜索空间的图搜索（HNSW）。

如何在十亿、百亿级特征库上做向量检索？主要挑战是索引无法放入内存，因而需要考虑内存-磁盘混合方案（SPANN）。

本文只介绍一小部分有效且正交的技术，这些技术组合在一起可形成当前的最佳实践。其他一些古典、经典技术可以参考本文引用的论文的背景综述。

# 相似度
首先回顾一下什么是**向量之间的相似度**:
- 对向量X和Y来说，x和y越接近，则x越大的地方y也大，x小的地方y也小，则内积越大。因此内积可用作相似度打分，不过不能抵抗尺度变化。
- 为了抵抗尺度变化，往往将内积正则化到[-1,1]，就变成了余弦相似度cosθ，θ为向量夹角，显然θ越小，向量越接近。
- 欧几里得距离正则化到[-1,1]后相当于sqrt(2-2cosθ)。可见欧式距离平方也是和余弦相似度成比的。

![sim_measure](https://cmbbq.github.io/img/sim_measure.png)

正则化后欧式距离、内积、cosθ三者同源，所以我们一般就用余弦相似度cosθ去度量向量/embedding之间的相似程度。除非又特殊的抵抗位置变化的需求，可采用皮尔森相关系数，皮尔森相关系数是协方差（两个变量的总体误差）的正则化，不过它同时也可以被视为将x和y中心化后的余弦相似度。

# 向量量化
量化器是D维向量 x  ∈ R^D 到 q(x) ∈ C = {c_0, c_1, ... c_k-1} 的映射函数q。c_i即聚类中心centroid。
所谓密码本（codebook）就是一个lookup table，用centroid下标作为原始向量的低bit表示，密码本大小就是k。

向量量化本质是有损压缩，表征学习本质上也是有损压缩，一个白盒，一个黑盒，都是试图降低高维数据的编码率。

# IVF：聚类、倒排、剪枝
倒排（特指IVF）是一种古老的量化技术，早期应用于[Video Google](https://www.robots.ox.ac.uk/~vgg/publications/2003/Sivic03/sivic03.pdf)。核心思想是基于k-means聚类做向量量化。k-means聚类起源于信号处理领域，目标是把n个向量分区到k个clusters，每个向量属于离centroid最近的cluster。

聚类后有些边缘数据点既属于A，也属于B，那就把它同时放到A和B的centroid对应的postinglist里面去，不过重复放置会导致postinglist膨胀，为了应对这种膨胀，可采取剪枝策略，具体可参考微软的SPTAG项目和[SPANN论文](https://arxiv.org/pdf/2111.08566.pdf)。

Faiss的IVFFlat就是在聚类形成centroid为term的倒排后提供检索能力，nlist为聚类数，nprobe为临近centroid探测数，nprobe=nlist时退化为暴搜。IVFFLat在检索时将query邻近的nprobe个centroid对应的所有原始向量作为搜索空间进行暴搜。由于centroid的表示方式可以简化成clusterid，所以实际上IVF也是一种量化：用一个数组存放所有的centroid向量，数组下标即可表示centroid。

# PQ：乘积量化
基于聚类的量化对低维向量比较有效，但随着向量维度升高，出现误差也变高，而且这种误差不是提升聚类数目能解决的。如果把聚类数目（codebook大小）提升到2^64，训练这个聚类的开销就会高得无法接受，需要数倍于2^64的样本，内存显然也放不下。**乘积量化（PQ）**就是降维手段解决IVF量化误差的技术，[Jegou et al., 2011](https://lear.inrialpes.fr/pubs/2011/JDS11/jegou_searching_with_quantization.pdf)以及[ScaNN](https://arxiv.org/pdf/1908.10396.pdf)均讨论了PQ。其核心思想是对索引数据量化，将d维空间切分为M个子空间，将高维向量的误差拟合为分段的低维向量误差之和。

乘积量化中，IVF倒排链存的不是原始向量，而是原始向量的一种编码：
- 求原始d维向量与高维粗聚类centroid的残差（残差的目的是中心化，让数据点分布变得更集中，使精度更好）
- 将残差向量切成M个分段，每个分段维度为d/M
- 每个分段做k*=2^n个细聚类，n即为低维细聚类clusterid的编码长度，4bit或8bit
- 用每个分段的邻近低维细聚类centroid去量化该分段上的原始残差分段

Faiss的IVFPQ就结合了IVF和PQ，IVF粗聚类是一级量化，PQ则是二级量化。一方面索引体积减小，另一方面计算上也只需要计算残差和分段细聚类centroid近邻的距离，还能SIMD加速，性能收益非常显著。不过IVFPQ的distance究竟只是一种拟合，是距离粗聚类centroid的距离加上M个分段里query残差和邻近细聚类centroid的距离总和，这种拟合可以一定程度上帮助找到近邻，但不适合用作距离，可以把IVFPQ当做粗排，结果再做一次brute-force或高精度ANN重算，得到更精确的距离。

ScaNN提出了各向异性量化，算是正本清源，纠正了此前就近选择centroid在数学上的错误，并且实践了4-bit量化取得非常好的效果（其实4bit量化的效果还比各向异性更大，但各向异性在理论上贡献更大）。传统的量化把数据点量化到离自己最近的聚类中心，但这么做未必是误差最低的，越平行的向量之间内积越大，越正交的向量之间内积越小，因此赋予平行向量更高的误差惩罚权重效果更好。这就意味着不一定量化到距离最近的centroid了，目标是整体编码率不变的前提下，让平行方向量化误差更小，正交方向误差高一些其实无所谓，从而提升整体的ANN效果。

假设M = 4，采用8bit量化，原始向量是256维f32向量，则PQ将256个float32压缩到64个int8，内存减少到1/16。乘积量化显然可以显著减少内存和存储开销，其实这也能加速计算，原因如下：
- 高效点乘（其实也是误差换效率）：O(dk+mn) 比 O(nd)快，d维度的query和n个量化后的向量点乘，引入m个大小为k的量化codebook，mn可以忽略不计，k远小于n。
- memory bandwidth：现代处理器需要“高计算量per memory read”才能充分发挥硬件的计算性能。量化压缩数据点后内存带宽的使用也下降了，变得更计算密集。
- 向量量化中引入的低bit表示，尤其是4bit量化，和低bit浮点数量化一样，可以享受SIMD的性能红利。

# 最佳实践：根据应用场景将各种正交技术进行正确组合
[HNSW](https://www.robots.ox.ac.uk/~vgg/publications/2003/Sivic03/sivic03.pdf)、IVF、PQ、各向异性的score-aware quantization loss、brute-force事实上都是正交的技术，可以组合起来使用。

比如进行了PQ量化的数据点可以再构建一个HNSW，用图方法在超大规模数据集上做查询显然比ivfpq更高效。各向异性量化则是对此前PQ量化的修正。考虑到量化后的距离失真，还能用brute-force把ivf+pq+hnsw的粗排结果重算一下得到最精确的距离，由于已经经过一轮粗排，候选数据集大小从十亿级别缩小到万级别，brute-force的开销就完全可以接受了。

