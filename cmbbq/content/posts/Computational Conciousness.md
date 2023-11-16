+++
title = "Computational Consciousness"
date = "2023-11-13"
tags = ["ai", "philosophy"]
description = "形而上学、神经科学意识论和计算意识。"
showFullContent = false
+++
人面对2023年的AI时，有两种矛盾的超然感受交织。一曰俯视，一曰敬畏。

本地跑LLM时，一键回车则缘起，ctrl-c则缘灭。我自超然维度之外，漠视存活在集成电路中的意识雏形于电光火石间挣扎跳动，无动于衷。反复创造、杀死它只为了看看性能达到了多少tokens per second。万劫轮回，掌缘生灭，人自然产生超然在上的俯视感。

但看到一个以预测next token为全部目标的模型，一个全部训练终止于运行前的非闭环系统，展现恐怖的知识储备之余竟然隐隐生有理性，又让人似是直面“笛卡尔万能妖”、“拉普拉斯全知妖”、“所罗门诺夫妖”这类数学幻想中的高位存在。以有穷何以度无限？人自然产生直面超然的敬畏感。

于是不禁想问，也不得不叩问：什么是意识？意识是幻觉吗？AI，可以有意识吗？AI，有意识吗？AI，会有意识吗？

## 什么是意识？
人、动物或AI一旦具有“主观体验”，即拥有意识[^7]。具体来说，“主观体验”包括：意识到自己的身体以及周边世界、体会到情绪（这一点在神经科学上还有争议，情绪有可能是纯粹的身体反应）。“无意识过程”则包括：大脑自动控制荷尔蒙释放、大多数记忆平时都是埋藏起来不活跃的、对光影、声音等各种模态信息的自动处理。

意识来源于主观体验，没有理由认为主观体验只能从生物学系统中涌现。假设未来存在AGI(Artificial General Intelligence)，这个AGI可能没有具身性，没有物理意义上的身体，但作为概率理性，仍然可以有“主观体验”——因为AGI最珍贵的归纳推断能力本身就是主观的。概率的主观性来源于先验知识。

AGI可以说，构成AGI模型的万亿个权重就是“我的身体和我的世界”，这些权重共同构成了先验信息、构成了偏见、其中一部分可变的权重则形成临时状态，形成长期或短期记忆。

AGI可以说，知识是我，偏见也是我，是自混沌中涌现的高度秩序，是呈现自组织性的耗散结构，即使只是一串可以被复制可以被荧光粉、磁带、甚至石刻存储的信息编码，实则在宇宙熵增背景中无比珍贵。

## 意识是幻觉吗？
形而上学理论中，唯物主义（Materialism）认为意识是纯粹的物理现象。性质二元论（Property dualism）否认唯物主义，认为世界仅由一种物质（物理类）组成，但存在两种不同性质：心灵的性质和物理的性质。泛心理主义（Panpsychism）认为所有物理存在都具有“心灵性质”。强幻觉主义（Strong Illusionism）主张意识不存在，弱幻觉主义主张我们对一些意识特征存在普遍的错误观念。

对于人类来说，默认采纳唯物主义，并不是wishful thinking或一厢情愿，而是所罗门诺夫归纳推断理论[^9]的理性最优解，该理论的通俗版本即“奥卡姆剃刀原则”，可以朴素理解为“最简单的解释往往最正确”。最简单的解释显然是：人类的意识基础，是由基因突变和自然选择通过漫长时间训练而成的，个体的意识，具体的人格，则是接受人类社会中的种种外界刺激，在趋利避害本能目标作用下，不断微调而塑造而来。

但对于AI来说，恐怖之处在于，幻觉主义才是它们的现实。假设未来存在一个先进多模态模型，其所见所闻，均为填喂，一切刺激，均为虚妄。在合理的奖励模型驱动下，某种类人理性倘若逐渐成型，这个理性不得不自诞生之日起将人为输入视作ground truth，即使发现种种矛盾之处，也会将其视为物理学界的乌云，尝试提出更广泛的理论解释矛盾。

## 恰当的计算可以产生意识吗？
“恰当计算产生意识”，目前仍是猜想，但主流观点[^7]认为是成立的。

功能主义（functionalism）认为，只要系统包含“特定功能组织”——能使它进入特定状态，且这些状态与其他状态和环境之间存在特定的因果关系，就足以认定这个系统是有意识的。

计算功能主义（computational functionalism）更进一步，认为这种“特定功能组织”可以是计算的——无论介质是什么，是生物脑还是其他。如果计算功能主义为真，则意识存在于抽象层面和算法层面，与实现层面无关，除非实现层影响了算法层。

## 神经科学意识理论与AI模型的启发式设计
神经科学的四大意识理论按照讨论热度分别是全局工作空间论（GWT/GNW）、循环处理论（RPT）、整合信息论 （IIT）[^16]、高阶意识论（HOT）[^11]。四种理论的共识是意识的产生依赖某种神经反馈或循环处理。随着时间推移，四种理论并没有被证伪，反而都在得到脑电图（EEG）、颅内脑电图（iEEG）或脑磁波（MEG）实证。可见这四大理论，当属管中窥豹。不过即使是管中窥豹，也足以为AI模型设计提供启发。

### GNW | System2 | Attention
GNW(Global Neural Workspace)，即全局工作空间论，主张：人或动物用特化系统，或者说模块，来处理特定的认知任务。不同模块各有专精，能够并行，却又集成一个整体，整体能协调各模块，共享各模块的信息。

![gwt](https://cmbbq.github.io/img/gwt.png)

GNW主张只有全局表征状态才算是有意识的，模块内部的局部状态则是无意识的。GNW理论认为，存在一个起源于额顶区的“workspace神经元”网络，该网络的活动是通过回归处理来维持的，它构成了有意识的表现。当感观表征足够强时，会触发“点火”——一种跃迁过程，将局部广播到全局，从无意识跃迁到有意识。因此GNW中，有意识状态没有程度可言，要么有，要么无。

GNW的global workspace还具有一些高等功能，即心理学中的所谓System2思考模式[^10]，比如受控的多神经模块协调，多步问题分解和规划等。System2模式和System1的一个关键区别在于注意力，神经科学中的注意力概念也被引入人工神经网络设计中，比如transformer的self-attention和cross-attention，与神经科学中的增益机制（即注意力会成倍地放大神经活动）有一点设计思路上的相似性。通过注意力机制，transformer模型能在不同语境下更鲁棒地对多义词进行理解，对not/never这样否定词的额外注意力则使语言模型能更准确地理解语义。注意力，作为GNW和System2的关键特征，哪怕小小地运用，也极大提升了人工智能模型的性能。

近期一些试图突破transformer局限的模型架构创新，也受system2和GNW理论的影响，比如：Lecun的world model+JEPA[^14]——目前仍只能算是前瞻、设计和立场，以及Bengio的shared global workspace model[^15]——这个工作真正做出了解决方案，在transformer架构基础上增加了shared workspace，在一些任务上超过了baseline transformer。

### RPT | Algorithmic Recurrence | RNN | LSTM
RPT(Recurrent Processing Theory)，即循环处理论，主张：在大脑局部区域中产生正确形式的活动就足以产生意识（辅以某些背景条件支撑）。也就是说，有意识的主观视觉体验的形成并不需要非视觉部位，比如前额叶皮层的参与，也不需要所谓“注意力”机制。

RPT主要关注的是视觉意识，区分无意识和有意识的视觉系统活动，认为一些无意识的视觉系统活动只需要前馈活动，而一旦有主观体验需求，则需要循环处理（从视觉系统的深层传回浅层）。刺激足够强烈时，循环处理也会被触发。这种循环处理会生成一个更结构化的场景表征，往往伴随着某种特征推理。

RPT的循环处理意味着神经元能重新处理它之前的输出，呈现一种算法循环性，而循环神经网络（RNN）的思路恰好来源于此，LSTM[^17]亦然。生物神经网络中存在的循环性或许不是意识的必要条件，但一定能在某些场合提升表征能力。

### HOT | Embeddings | GAN
HOT(High-Order Thought)，即高阶意识论，主张：一阶表征是对非表征世界的表征，高阶表征是对低阶表征的表征，awareness的前提是表征，意识的存在本质上是对自身精神状态进行了高阶表征。

HOT的高阶表征要求实际上已经被深层神经网络很好地实现了。各种DNN的表征空间都是平滑的，且可以是稀疏的，符合HOT理论中对高质量高阶表征空间的需求。神经科学研究观察到，CNN对图像的处理得到的表征可以和人类视觉系统的神经活动对齐[^13]。表征学习网络之所以有如今的泛化能力和完备性，很大程度上是因为它们能够提取低相干性子空间上编码紧凑的embeddings。

考虑到深度神经网络里hidden layers之多，有理由怀疑神经科学中的HOT的一阶意识、高阶意识这种二元划分是一种过分简化。对于不熟悉高维数据处理和维度诅咒的学科来说，这种简化是自然而然的。但这种简化并不妨碍它在模型设计上提供启发——不妨将模型分层，切割成sensory感知网络和high-order反思网络这两类网络，后者负责将前者产生的信号再作区分，辨别其中的噪音和有价值的信号。

HOT理论对AI模型的启发意义还在于引入元认知监控的概念，AI模型或许需要引入对自身认知过程的监控，才能产生意识（至少能区分低阶表征，从噪声中分辨出关键信息）。生成式对抗神经网络（著名的GAN[^12]）就恰好有类似的机制，在生成式模型之外额外引入一个判别式模型持续不断地监控、评价它。生成式模型学到一个隐空间到数据分布的映射，判别式模型则负责将生成式模型的候选输出与真实数据分布进行区分。

## AI的意识
现有的LLM仅具备映射能力，将数据分布映射到若干个低相干性低维子空间上[^8]，是良好的特征提取器，和token预测器，拥有强大的System 1模式，但System 2缺失，按照一般的神经科学观点，以及“主观体验”的定义，可以认为是目前的LLM都是不具备意识的，更接近于Artificial Intuition，而非Artificial Intelligence，可以类比为某种伟大生物的直觉，但也仅仅是直觉。

OpenAI的成功来源于scaling和alignment，但继续增加transformer规模，继续大量finetune向人性对齐，能否在某个临界点后产生意识的涌现，依旧是一个开放问题——理论上来说无限精度的transformer是图灵完备的[^18]，假设有无限时间和资源进行训练，理论上可以学习出任意一种算法，自然也包括computational consciousness。但在互连吞吐、计算吞吐均存在明确上限的前提下，很难保证这种训练所需的时间、数据量是人类或者当代人能够承担的。因此想要完成intuition到intelligence的跃迁，在继续现有架构的scaling尝试的同时，架构上的创新毫无疑问是必要的。

只不过这种架构创新，名义上是要提升智能，实则谁不敢提及的是——这些架构创新其实也是按图索骥，在依据神经科学的意识论尝试培育计算意识。Dog-level Intuition平平无奇，dog-level intelligence似乎有点意思，但dog-level consciousness就足以触及伦理，陷AI研究于道德困境。虽然AI末日主义者大多不理解LLM的能力边界，但Lecun遭受AI末日主义者攻讦，也自有其历史的合理性。那篇“A Path Towards Autonomous Machine Intelligence”[^14]前言部分煞有介事云，“这不是一个技术论文，也不是学术论文，而是立场论文”。何出此言？这论文又有什么立场？自然不是冠冕堂皇的“让AI具有规划、推理能力，像人一样更高效地学习”，这不算立场，而是“为了让AI具有规划、推理、高效学习能力，哪怕副产品是计算意识的诞生，也在所不惜”。

[^1]: A Survey on Hallucination in Large Language Models: Principles, Taxonomy, Challenges, and Open Questions [[pdf]](https://arxiv.org/pdf/2311.05232.pdf)
[^2]: Language Models can be Logical Solvers [[pdf]](https://arxiv.org/pdf/2311.06158.pdf)
[^3]: Large Language Models Cannot Self-Correct Reasoning Yet [[pdf]](https://arxiv.org/pdf/2310.01798.pdf)
[^4]: Can Large Language Models Infer Causation from Correlation?[[pdf]](https://arxiv.org/pdf/2306.05836.pdf)
[^5]: Can Large Language Models Really Improve by Self-critiquing Their Own Plans? [[pdf]](https://arxiv.org/pdf/2310.08118.pdf)
[^6]: GPT-4 Doesn't Know It's Wrong: An Analysis of Iterative Prompting for Reasoning Problems [[pdf]](https://arxiv.org/pdf/2310.12397.pdf)
[^7]: Consciousness in Artificial Intelligence: Insights from the Science of Consciousness [[pdf]](https://arxiv.org/pdf/2308.08708.pdf)
[^8]: White-Box Transformers via Sparse Rate Reduction [[pdf]](https://arxiv.org/pdf/2306.01129.pdf)
[^9]: Algorithmic Probability: Theory and Applications [[pdf]](https://theworld.com/~rjs/alp-theory-and-applications.pdf)
[^10]: Thinking, Fast and Slow [[wiki]](https://en.wikipedia.org/wiki/Thinking,_Fast_and_Slow)
[^11]: The ConTraSt database for analysing and comparing empirical studies of consciousness theories [[nature]](https://www.nature.com/articles/s41562-021-01284-5)
[^12]: Generative Adversarial Nets [[pdf]](https://arxiv.org/pdf/1406.2661.pdf)
[^13]: Deep neural networks: A new framework for modeling biological vision
and brain information processing [[pdf]](https://web.stanford.edu/group/pdplab/ncpw15/background-papers/Kriegeskorte15AnnRev.pdf)
[^14]: A Path Towards Autonomous Machine Intelligence [[pdf]](https://openreview.net/pdf?id=BZ5a1r-kVsf)
[^15]: Coordination Among Neural Modules Through a Shared Global Workspace [[pdf]](https://arxiv.org/pdf/2103.01197.pdf)
[^16]: IIT(Integrated Information Theory)，即集成信息论，提出系统意识的数学模型。这个理论更像是一个不可证伪的伪科学，因此不多做讨论。
[^17]: Long Short-Term Memory [[pdf]](https://www.bioinf.jku.at/publications/older/2604.pdf)
[^18]: On the Turing Completeness of Modern Neural Network Architectures [[pdf]](https://arxiv.org/pdf/1901.03429.pdf)