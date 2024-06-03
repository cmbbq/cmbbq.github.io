+++
title = "Revisiting Recommender Systems"
date = "2024-06-03"
tags = ["ai"]
description = "本文是对孙爱欣教授的分享——“推荐系统研究现状的理解”的笔记，对大致内容进行了摘要，并收集了提及文献的链接——可以一窥推荐系统领域学术研究的现状，精辟，有趣，令人震撼。"
showFullContent = false
+++

推荐问题可定义为在合适的时间把合适的物品（items）推荐给合适的用户。

通常来说，推荐任务所用数据集即用户-物品交互矩阵。推荐系统基于这个数据集推测用户兴趣，把结果放到线上进行测试（学术界没有这个条件，只能线下评测），进行评测。

## What's Missing in User-Item Interaction Datasets
> 我们通常将推荐任务理解为如何在一个静态的用户-物品交互矩阵中预测缺失的数据，而不是在特定场景中的动态环境下预测用户的下一次交互。

值得注意的是，在用户-物品交互矩阵数据集中，时间的维度坍缩了，真实互动场景中的种种约束也坍缩了。

推荐系统文章中，70%使用了MovieLens数据集，但它真的能还原推荐场景吗？未必，因为MovieLens在一次初始化过程中完成用户对过去多年观影体验的回忆——可能有很多遗漏、遗忘，也忽略了上映时间、价格等现实因素。然而现实场景中用户的兴趣是逐渐形成的，受制于各种现实约束，其观影决策还不可避免地受到上映时间、票价、以及兴趣是否发生变化等诸多因素影响。

## A Worrying Analysis of Current Practice 
> 基于我们过去五年在推荐系统评测和数据集分析方面的工作，我们重新审视推荐系统的问题定义，并为缺乏共识的现象提供一种解读。

Dacrema et al., 2019[^1]认为深度学习新模型的效果一般般。

Bauer et al., 2024[^2]做了一个综合，分析了数据集，发现用的数据集非常集中，集中在MovieLens、Amazon Reviews等主流数据集，这些数据集普遍偏旧。大部分文章都是提出新的模型。还有一些专注在评测标准。

Ivanova et al., 2023[^3]认为推荐领域用哪个baseline其实并没有共识，一部分原因是每年推荐系统里有5k篇文章，没人能读完所有文章，于是审稿者之间未形成共识，有的审稿者认为某些方法特别好，一定要纳入baseline，另一些审稿者则有不同观点。比如nearest neighbor虽然简单，经过良好调参后却是非常强的baseline，在很多场景比复杂模型表现得好很多，但很多文章不会把nearest neighbor列入baseline。大家都认为它是几十年前的方法，不值得比较。

不仅baseline不统一，即使baseline统一，也有一个调参的问题，如Shehzad, Jannach, 2023[^4]所说，当你提出自己的模型时，非常注重调参，和别人比时却没有非常精细地调参。

McElfresh et al., 2022[^5]做了非常大规模的调研，在85个数据集上比较了24个算法的315个指标，得出了令人震撼的结论：这些算法并不能泛化，在某个数据集上很好，下一个可能就不好。每个算法都可能排第一第二，最差都能排到倒数第几。最后发现最强的算法竟然是Item-kNN！


## Data Leakage in Train/Test Split
Sun, 2022[^6]中对20~22年88篇RecSys会议文章的``train/test split``做了一个梳理，发现34%的论文用的是``random split``（随机分），25%用的是``leave-one-out``（最后一个交互做成测试，之前的是训练），19.5% ``single time point``，17% ``simulation-based online``，4.5% ``sliding window``。理论上最完美的train/test split方法是严格遵循时间线，在时间线上取某个时间点之前的做训练集，之后的做测试集，然后慢慢把这个时间点往后推，每个时间点的选取生成了相应的训练集和测试集，时间点越往后推，训练集越多测试集越少——可惜实际操作起来很难做到，大部分文章采用的``random split``和``leave-one-out``有很强的信息泄露。

以``leave-one-out``为例，每个用户都只取最后一次交互做测试集，问题就在于不同用户的最后一个交互的时间可能大不同——假设某个物品在特定时间非常火，``popularity``(流行度是推荐系统中最简单的baseline，就是对物品的交互次数进行排序)排序非常高，但某个用户最后一次交互时间甚至在这个特定时间之前，那给这个用户推荐一个未来爆火的物品显然并不合理。Ji et al., 2020[^7]就重新审视了这一问题，将``popularity``修正为用户最后一次交互时间点的“当时流行度”后，``popularity``置信度可以提升70%。

在Ji et al., 2020[^8]的研究中指出：几乎所有的ML/DL模型中，尤其是离线评测的推荐系统模型中，都存在类似的数据泄露问题——模型无意中使用了未来数据进行训练，学到了本不应该存在的用户-物品交互。``BPR``、``NeuMF``、``LightGCN``、``SASRec``均未从机制上避免这种数据泄露。这个研究也通过实验证明了这种数据泄露确实会显著影响模型的推荐准确率，且这种对准确率的影响是不可预测的。

## Recommendation should be Task-specific & Dynamic
用户决策涉及通用偏好和当下context因素。当下context是非常task-specific且非常动态的。这也使得推荐任务具备了这一特征。

很大程度上，现有的推荐系统都只局限在通用偏好层面。回顾数据层面，现有数据集，即各种用户-物品交互矩阵，显然丢掉了context，只能捕捉通用偏好。模型层面，训练是基于决策结果的，而非决策过程本身，这也决定了模型只能学到用户的通用偏好。而评测端，自然而然也只能对通用偏好进行评测。

工业实践中，推荐系统应该是一个检索问题。在这个检索问题中，query包含通用偏好、当前context这两种动态更新的信息；item collection也是动态更新的；ranking则旨在提升决策质量。

考虑到context因素，不同场景的推荐系统，如食物推荐、电影推荐、电商推荐、宾馆推荐，也应该有非常不同的实现，分开建模。有的场景选项固定，有的场景就是适合重复，有的场景则偏重探索。不同场景的输入甚至也有区别。比如外卖推荐除了user id之外，必需提供收货地址、早餐/午餐/晚餐这种信息。

## Conclusions and What's Next
学术界把具体工程问题简化、抽象，把具体问题的ground-truthing也连带着简化掉，一时间百花齐放，极度繁荣，实则无一堪用，深度模型还没真正击败kNN，今天各种生成式推荐又开始拥抱LLM。后世回顾今日，恐怕又是一例百万漕工。

孙教授近期的一篇文章Sun, 2024[^9]，对推荐系统的问题定义进行了重新思考，认为当前的推荐系统研究对推荐问题做了过度简化，以至于学术界提出的几乎所有的方案都不太适应现实世界中的具体任务。对未来的研究方向进行了一些研判：
- 预计不会有赢家通吃的模型，未来依旧是每个模型在自己的论文里无敌。
- 应将推荐系统问题细化，短视频有短视频赛道，电商有电商赛道，新闻有新闻赛道，针对不同赛道，设计、评测新的模型。别再用MovieLens去评测电商推荐了！
- Item-kNN仍然会是很强的baseline。只不过对nearest和neighbor的定义需要更好的特征工程。

[^1]: Are we really making much progress? A worrying analysis of recent neural recommendation approaches [[arxiv]](https://arxiv.org/abs/1907.06902)
[^2]: Exploring the Landscape of Recommender Systems Evaluation: Practices and Perspectives [[pdf]](https://arxiv.org/pdf/2311.05232.pdf) 
[^3]: RecBaselines2023: a new dataset for choosing baselines for recommender models [[arxiv]](https://arxiv.org/abs/2306.14292)
[^4]: Everyone’s a Winner! On Hyperparameter Tuning of Recommendation Models [[pdf]](https://dl.acm.org/doi/pdf/10.1145/3604915.3609488)
[^5]: On the Generalizability and Predictability of Recommender Systems[[arxiv]](https://arxiv.org/abs/2206.11886)
[^6]: Take a Fresh Look at Recommender Systems from an Evaluation Standpoint [[arxiv]](https://arxiv.org/abs/2210.04149)
[^7]: A Re-visit of the Popularity Baseline in Recommender Systems [[arxiv]](https://arxiv.org/abs/2005.13829)
[^8]: A Critical Study on Data Leakage in Recommender System Offline Evaluation  [[arxiv]](https://arxiv.org/abs/2010.11060)
[^9]: Beyond Collaborative Filtering: A Relook at Task Formulation in Recommender Systems [[arxiv]](https://arxiv.org/abs/2404.13375)