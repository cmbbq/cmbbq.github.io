+++
title = "Artificial Intuition: Reasoning Abilities of LLMs Today"
date = "2023-11-19"
tags = ["ai"]
description = "对近期对LLM的调研文献进行梳理总结，讨论目前大语言模型架构的推理能力边界。"
showFullContent = false
+++

## System1 vs System2
System1和System2的划分来源于心理学家Daniel Kahneman的理论[^8]，描述了思考的两种模式，System1指的是快速、自动、直觉且不费力的模式，System2则是慢速、刻意、分析性且有意识的模式。

现有的LLM除了已被证明的“将数据分布映射到不相干低维子空间[^7]的反射式推理能力”(System1)，是否具有一定的“慢速思考”、多步推理和规划能力(System2)?

[^1]: A Survey on Hallucination in Large Language Models: Principles, Taxonomy, Challenges, and Open Questions [[pdf]](https://arxiv.org/pdf/2311.05232.pdf) 
[^2]: Language Models can be Logical Solvers [[pdf]](https://arxiv.org/pdf/2311.06158.pdf)
[^3]: Large Language Models Cannot Self-Correct Reasoning Yet [[pdf]](https://arxiv.org/pdf/2310.01798.pdf)
[^4]: Can Large Language Models Infer Causation from Correlation?[[pdf]](https://arxiv.org/pdf/2306.05836.pdf)
[^5]: Can Large Language Models Really Improve by Self-critiquing Their Own Plans? [[pdf]](https://arxiv.org/pdf/2310.08118.pdf)
[^6]: GPT-4 Doesn't Know It's Wrong: An Analysis of Iterative Prompting for Reasoning Problems [[pdf]](https://arxiv.org/pdf/2310.12397.pdf)
[^7]: White-Box Transformers via Sparse Rate Reduction [[pdf]](https://arxiv.org/pdf/2306.01129.pdf)
[^8]: Thinking, Fast and Slow