+++
title = "Artificial Intuition: Reasoning Abilities of LLMs"
date = "2023-11-29"
tags = ["ai"]
description = "对近期对LLM的调研文献进行梳理总结，讨论目前大语言模型架构的推理能力边界。"
showFullContent = false
+++

## LLM具有System2吗？
System1和System2的划分来源于心理学家Daniel Kahneman的理论[^8]，描述了思考的两种模式，System1指的是快速、自动、直觉且不费力的模式，System2则是慢速、刻意、分析性且有意识的模式。

现有的LLM除了已被证明的“将数据分布映射到不相干低维子空间[^7]的反射式推理能力”(System1)，是否具有一定的“慢速思考”、多步推理和规划能力(System2)?

LLM横空出世之初，有很多狂野声明，比如"LLM as zero-shot planner"，"LLMs are zero-shot reasoner"，但这些声音在现在看来更像是一时跟风和hype。

现在，无论是普通从业者，还是知名学者，如Yann Lecun，Yoshua Bengio，马毅，Subbarao Kambhampati逐渐形成共识，认为LLM不具备System2，很多研究也通过一些reasoning benchmark对这一观点进行佐证，比如Huang et al., 2023[^3]，Jin et al., 2023[^4]，Valmeekam, Marquez, 2023[^5]，Stechly et al., 2023[^6]，Dziri et al., 2023[^12]。

也有少数学者，比如Ilya Sutskever在访谈中说scaling足以产生System2能力，无需架构创新，但Ilya似乎并不坦诚，动机不明[^11]。有研究者认为LLM加上Chain-of-Thought prompting是具有推理能力，如Saparov & He, 2023[^9]，Feng et al., 2023[^2]，但终究依赖了外界的prompting，且实验方法上并不能排除近似信息提取模拟了逻辑推理的可能。

LLM本身，考虑到transformer每次生成回答计算量是有上确界的，注定不可能为某个问题倾注不成比例的计算量，仅此一点，就可以从理性层面否定LLM具备Human-like System2。毕竟System2从定义上，就是一个长期保持专注的慢思考模式，理应具有潜在无限时间的思考能力。

## 假设System1足够强，能替代System2吗？
理论上来说，无限的精度的transformer都不需要无限权重，已被证明是图灵完备的[^10]，也就是说只要以恰当的方式学习，就足以学习到任何算法。

假设未来的LLM能具有近乎无限的精度、权重，并用超强算力做训推，则确实可能以System1模拟出System2的推理能力，用简单的直觉映射模拟演绎推理。但考虑到我们现在用作推理的LLM精度已经低到8bit，甚至4bit，即使如此成本都已经难以维持，有理由认为这种思路是不切实际的，这种架构所需的能耗和物质基础，轻而易举就能超出人类物质文明极限好几个数量级。

强无敌的System1能在刹那间把“意识”制造出来吗？在穿过多层transformer时意识萌发，再让意识消亡于logits的softmax。无限精度假设下似乎也不是不可能，毕竟能模拟一切算法，模拟出一种System2也不足为奇，那就相当于在一次执行过程中，制造了意识/生命的虚拟机，哪怕真正执行它的物理机只是朴素的transformer——一种纯粹以提升预测下个token似然为目标的自回归模型。这也是Ilya认为scaling足以产生AGI的理论依据。

## Artificial Intelligence？更像是Artificial Intuition
总而言之，将LLM称为人工智能仍然是夸大其词的，它从原理上讲就是一种人造直觉。这个直觉或许可以在无限算力无限精度无限权重假设下模拟出近人智能，但诚实的Datacenter AI从业者会承认—现有大模型在尺度上已接近了先进计算和先进互连的硬件极限，而相比算法突破，硬件的发展曲线是平缓且存在上确界的。

[^1]: A Survey on Hallucination in Large Language Models: Principles, Taxonomy, Challenges, and Open Questions [[pdf]](https://arxiv.org/pdf/2311.05232.pdf) 
[^2]: Language Models can be Logical Solvers [[pdf]](https://arxiv.org/pdf/2311.06158.pdf)
[^3]: Large Language Models Cannot Self-Correct Reasoning Yet [[pdf]](https://arxiv.org/pdf/2310.01798.pdf)
[^4]: Can Large Language Models Infer Causation from Correlation?[[pdf]](https://arxiv.org/pdf/2306.05836.pdf)
[^5]: Can Large Language Models Really Improve by Self-critiquing Their Own Plans? [[pdf]](https://arxiv.org/pdf/2310.08118.pdf)
[^6]: GPT-4 Doesn't Know It's Wrong: An Analysis of Iterative Prompting for Reasoning Problems [[pdf]](https://arxiv.org/pdf/2310.12397.pdf)
[^7]: White-Box Transformers via Sparse Rate Reduction [[pdf]](https://arxiv.org/pdf/2306.01129.pdf)
[^8]: Thinking, Fast and Slow
[^9]: Language Models Are Greedy Reasoners: A Systematic Formal Analysis of Chain-of-Thought [[pdf]](https://openreview.net/pdf?id=qFVVBzXxR2V)
[^10]: On the Turing Completeness of Modern Neural Network Architectures [[pdf]](https://arxiv.org/abs/1901.03429)
[^11]: 不负责任的猜想：考虑到OpenAI在尝试Q*这样的架构突破，Ilya在访谈时很可能存在故意误导的动机，不愿透露OpenAI的研究思路。此外强调scaling有利于凸现其个人的历史贡献，夸大AGI危机也有利于其负责的super alignment项目。
[^12]: Faith and Fate: Limits of Transformers on Compositionality [[pdf]](https://arxiv.org/abs/2305.18654)