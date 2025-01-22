+++
title = "Recommender Systems"
date = "2025-01-22"
tags = ["ai", "sys"]
description = "借助DeepSeek R1读论文，梳理推荐系统。"
showFullContent = false
+++

## 搜推架构概述
传统搜推采用三层漏斗结构(matching召回/preranking粗排/ranking精排)。

召回阶段兼顾相关性和一定的排序能力，往往多路召回。
- 传统三层结构中，召回和排序的分离导致商业回报较低，因此召回时可通过相关性模型做训练样本增强[^1]，或将相关性badcase作为负反馈建模[^2]。
- 样本空间设计需考虑SSB问题[^7]：
    - 做in-batch负采样[^8]和logQ流行度纠偏[^9]。
    - 构建中等难度负样本[^10]。
    - 可参考Pinterest广告检索的MUDA（Modified Unsupervised Domain Adaption）方法[^11]。
- 模型结构：朴素双塔、MVKE[^12]、QARM[^13]。

粗排阶段在相关性有保障的前提下，主要考虑排序能力，筛选出优质结果。
- 粗排同样需要应对SSB。[^15]
- 除了传统粗排模型外，还可以考虑和精排对齐的方法：COPR[^16]。

精排阶段的目标是通过精细化建模与多维度优化，从候选集中选出最优内容，精准预测用户行为，最大化某种商业目标。
- 精排：精细化打分百级候选（DeepFM、DIN、MMOE），侧重精准预测（CTR/CVR）和特征交互。
- 重排序：调优Top结果（PRM、DPP、MMR），侧重列表级优化（多样性、上下文）。

## Mobius
Mobius项目旨在通过将CPM[^3]、CTR[^4]、CPA[^5]、ROI[^6]等商业指标整合到匹配层，同时保持低延迟和计算资源限制，从而提升效果。

Mobius有两项关键技术：主动学习的CTR模型和快速广告检索。前者通过教师-学生框架，利用已有相关性模型生成合成数据，增强CTR模型对长尾查询和广告的泛化能力。后者就是我们做内容检索方向非常熟悉的ANNS/MIPS和OPQ向量压缩等检索技术。

Mobius中，CTR模型是直接集成到了召回阶段（传统搜推的CTR往往在排序阶段，和召回割裂）的双塔DNN（query塔+ad塔），在召回时就同时考虑相关性和商业价值，对数十亿的广告候选也能做出高效筛选。

ANNS不必赘述，OPQ（Optimized Product Quantization）是一个新颖的高维向量低损压缩机制，对PQ稍作升级，对原始向量施加了正交矩阵R，使变换后的向量在子空间中分布更均匀，减少子向量之间的相关性，从而减少了PQ量化误差（PQ将向量分割成多个子向量，这些子向量之间可能存在相关性）。

## "Not-to-Recommend" Loss
Google在[^2]中提出一个利用用户负反馈进行训练的"Not-to-Recommend" loss，显式地利用负反馈信号优化不推荐该项目的log-likelihood。这个loss和标准的正反馈交互交叉熵loss合并，形成一个兼顾正、负反馈的联合学习机制。

此前大多数利用负反馈的工作往往是ad-hoc的特征工程。

## SimANS 
SimANS是针对密集文本检索任务剔除的负样本采样方法，旨在解决现有方法中存在的非信息性负样本（过于简单）和错误负样本（过于困难）问题。其核心思路是通过选择模糊负样本（ambiguous negatives）——即与正样本相关性分数接近的负样本，平衡样本的难度与信息量，从而提升模型训练效果。

其创新点在于首次明确将中等难度负样本（ambiguous negatives）作为核心采样目标，提出其梯度特性（均值大、方差小）的理论依据，并通过实验验证其对模型训练的重要性。

此外，SimANS引入了基于指数函数的概率分布，通过分数差异动态调整采样权重，避免固定策略（如随机或Top-k）的缺陷，同时兼顾计算效率。

## MUDA
传统推荐系统在训练数据中存在选择偏差，训练数据分布往往和实际推理时的数据分布不一致。广告检索领域，训练数据主要来自后续阶段（拍卖胜出者），这些数据是多轮筛选后的结果，和真实全量候选广告集有很大分布差异。

传统的基于用户行为的建模忽视了大量未被选中广告的信息，MUDA[^11]将精排结果（即伪标签）转化为二元标签（正负例），用二元交叉熵loss替代回归损失，避免过拟合低置信度的伪标签，从而在训练中间接利用到哪些未被标记却与真实推理时数据分布更接近的数据。

## MVKE
此前用户画像建模普遍使用多个独立双塔模型预测CTR和CVR[^14]，但存在数据稀疏（单一动作模型，比如仅点击、仅转化无法互补学习，导致数据稀疏动作的建模效果差）、特征交互不足（双塔模型两塔分离，用户和标签的交互不足，难以捕捉多主题偏好）的问题。

MVKE模型中设置了虚拟内核专家（VKE）和虚拟内核们（VKG门控结构）联合学习用户在不同动作（点击、付费）和主题（体育、汽车）上的偏好，提升用户画像多样性和准确性。MKVE中一个用户塔包含多个VKE，每个VKE负责用户偏好的一个特定领域，并通过所谓虚拟内核（某种可学习参数）指导特征交互。VKG根据标签embedding和虚拟内核动态融合不同VKE的输出。

在“点击->转化”序列中，点击作为浅层任务，可以为转化这一深层任务提供基础信息。

## QARM
QARM是基于MLLM（多模态大语言模型）的推荐架构。论文指出工业界推荐系统的MLLM有两个问题，一个是表示不匹配，一个是表示无法学习。前者指的是预训练的多模态模型与下游推荐任务的目标不一致（比如说图片表征本来是用来做image-text匹配的），后者指的是多模态表示在推荐模型中无法通过梯度更新进行优化（往往MLLM生成的表征只用做推荐模型的额外输入）。

QARM框架提出了Item Alignment（项目对齐机制）和Quantitative Code（量化编码机制）。
- 对齐：通过业务特定的用户-项目交互数据（如高质量Item2Item对），微调预训练多模态模型，使表征与下游任务对齐。例如，针对电商场景调整因果商品关系，而非通用语义对齐。
- 量化：将对齐后的多模态表征转换为可学习的离散编码ID（如VQ向量量化与RQ残差量化），支持端到端训练。编码ID作为推荐模型的输入特征（如用户兴趣序列、物品属性），替代静态表征。

## ASH & ASMOL
传统粗排往往被视为迷你精排，用轻量模型，依赖AUC等有序列表评估指标，但这种方法的离线评估和在线A/B测试结果不一致。而且粗排的目标其实不是生成有序列表，而应该是高质量无序候选集。

论文[^15]认为盲目模仿精排模型会限制粗排潜力，需专注集合质量而非单纯一致性。

论文引入ASH(All-Scenario Hitrate)指标，将推荐、购物车等不同场景的购买正样本都放在一起，缓解单一场景样本偏差，更全面地衡量粗排候选集质量。实验证明ASH与在线GMV强相关，优于传统指标AUC(Area Under the ROC Curve)[^17]、ISPH(In-Scenario Purchase Hitrate)。

论文还提出来多目标学习框架ASMOL：
- 全空间训练样本：整合曝光样本、排序候选样本、预排序候选样本，覆盖更全面的数据分布。
- 多目标损失：联合优化曝光、点击、购买任务，学习优先级（购买 > 点击 > 曝光）。
- 知识蒸馏：从排序模型蒸馏知识，但仅针对曝光样本（避免噪声干扰）。
- 单模型架构：统一模型处理多任务，优于传统多模型策略。

## COPR
COPR[^16]察觉到相对轻量的粗排模型的能力不如复杂精排模型，因此以对齐精排结果为目标建模，通过分块采样和排序对齐，将目标从分数对齐放宽为排序对齐，有效缓解了模型容量差异与投标误差放大的问题。其即插即用设计可适配多种预排序模型，并在淘宝广告系统中成功落地，显著提升了广告效果与平台收益。该研究为级联架构中的模型协同优化提供了新思路。

COPR缺乏探索策略。分块采样可能保留不同优先级的广告组，ΔNDCG加权更关注头部广告的排序正确性，这主要影响排序的准确性而非多样性。如果头部广告本身多样性不足（如重复推荐同类高CTR广告），则可能引发信息茧房。



[^1]: MOBIUS: Towards the Next Generation of Query-Ad Matching in
Baidu’s Sponsored Search. [[pdf]](https://arxiv.org/pdf/2409.03449)
[^2]: Learning from Negative User Feedback and Measuring
Responsiveness for Sequential Recommenders. [[pdf]](https://arxiv.org/pdf/2308.12256) 
[^3]: CPM(Cost per Mile)为千次广告展示的成本，适用于品牌曝光。
[^4]: CTR(Click-through Rate)为用户点击率，适用于搜索广告。
[^5]: CPA(Cost per Action)为每次用户行为成本（入下载、购买），适用于深度目标转化。
[^6]: ROI(Return on Investment)为投资回报率，收益和成本的比值。
[^7]: SSB(Sample Selection Bias)是指训练数据中样本选择过程不随机或不符合真实数据分布，影响模型效果的问题。推荐系统中，热门项目往往占据大部分用户交互数据（数据分布highly skewed才是常态），长尾项目（冷门项目）的数据过于稀疏，若模型过度依赖热门项目的数据，则容易导致对长尾项目的推荐效果不佳。负样本（用户未交户的项目）若随机采样，很容易导致占比高的热门项目倍过度采样为负样本，又会对热门项目不公平。此外，训练数据若来自特定时间段，而测试数据又来自另一时间段，则可能导致对历史时期的过拟合，无法适应当下潮流。
[^8]: In-batch负采样是简化负样本采样的方法，对当前训练批次中每个正样本（user-item pair）之外的其他项目都作为负样本。这样做可以直接从当前批次获取负样本，无需额外采样操作，也不必维护一个全局负样本池，降低计算开销（大规模推荐系统的项目太多，每次训练都从整个数据集随机采样负样本非常耗时，也容易导致热门项目倍过度采样为负样本）。这么做也有缺点，比如负样本多样性不足。
[^9]: 估计项目的流行度（被采样的概率）并用logQ修正模型得分，减少负采样中对热门项目的过度惩罚，详见[Sampling-Bias-Corrected Neural Modeling for Large Corpus Item Recommendations](https://dl.acm.org/doi/abs/10.1145/3298689.3346996)。这里的logQ correction受启发自[sampled softmax model](https://www.iro.umontreal.ca/~lisa/pointeurs/importance_samplingIEEEtnn.pdf)。
[^10]: 简单in-batch随机负采样生成的非信息性负样本往往过于简单，更均衡的负采样方法参考：SimANS: Simple Ambiguous Negatives Sampling for Dense Text Retrieval. [[pdf]](https://arxiv.org/pdf/2210.11773)
[^11]: An Empirical Study of Selection Bias in Pinterest Ads Retrieval. [[pdf]](https://dl.acm.org/doi/pdf/10.1145/3580305.3599771)
[^12]: Mixture of Virtual-Kernel Experts for Multi-Objective User Profile Modeling. [[pdf]](https://arxiv.org/pdf/2106.07356)
[^13]: QARM: Quantitative Alignment Multi-Modal Recommendation at Kuaishou. [[pdf]](https://arxiv.org/pdf/2411.11739)
[^14]: CVR(Conversion Rate)：转化次数和点击次数的比值。
[^15]: Rethinking the Role of Pre-ranking in Large-scale E-Commerce Searching System. [[pdf]](https://arxiv.org/pdf/2305.13647)
[^16]: COPR: Consistency-Oriented Pre-Ranking for Online Advertising. [[pdf]](https://arxiv.org/pdf/2306.03516)
[^17]: ROC(Receiver Operating Characteristic Curve)曲线即受试者工作特征曲线，是一种用于评估二分类模型性能的工具，以真阳率TPR为纵轴、假阳率FPR为横轴，对每一个阈值，计算对应的（FPR, TPR）坐标点，将这些点按阈值从高到低连接起来，即形成ROC曲线。AUC即ROC曲线下面积，AUC=1即为完美分类器，AUC=0.5为随机猜测。
[^18]: ISPH@k，即场景内购买命中率，衡量的是粗排输出的前k个候选项中是否包含用户实际购买的商品。局限性在于：(1)k等于粗排输出的全部候选数时，ISPH@k恒为1，就失去评估意义了。（2）粗排候选集在精排后才能曝光，ISPH@k反映的是整个排序阶段的选择，而非粗排候选集本身的质量。（3）仅捕捉单个场景内的偏好。
