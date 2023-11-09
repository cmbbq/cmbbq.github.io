+++
title = "Local LLM"
date = "2023-11-02"
tags = ["ai", "sys"]
description = "用llama.cpp在本地cpu上跑ggul格式的mistral7b的简明教程。"
showFullContent = false
+++

## Minimalist Local LLM
新一轮AI热潮中，stable diffusion和LLM均呈现出了前所未有的社区繁荣。繁荣的同时带来了信息的爆炸和臃肿，极简主义显得尤为珍贵。

在预训练模型中，极简主义的代表是小而美的“大模型”Mistral7B。在推理框架中，极简主义的代表是对抗各种大公司重量级项目的llama.cpp。

Mistral7B是开源模型领域的新近进展，模型虽小，却展示了惊人的能力。根据Mistral AI自己的benchmark和一些社区的测试反馈，几乎在各方面击败llama2 13B。

![mistral](https://cmbbq.github.io/img/mistral.png)

Mistral AI提供了在本地GPU、云端跑Mistral7B的官方支持。

不过这么小的模型，其实也可以在本地CPU上跑的——借助llama.cpp。

llama.cpp和whisper.cpp大体上是Georgi Gerganov的个人项目，基于自己写的ml primitive库ggml实现，用最少的代码量实现了尽可能多的推理优化，针对Apple芯片做得尤其好，也适用于Linux服务器，适用于有或无GPU，单或多GPU的硬件setup。ggml规定的模型文件格式ggul甚至也在社区中被广泛使用，成为事实上的标准之一。

## Intel Xeon Platinum 8336C + Debian10
1. 从TheBloke上下载特定量化版本的模型：https://huggingface.co/TheBloke/Mistral-7B-Instruct-v0.1-GGUF
2. ```git clone https://github.com/ggerganov/llama.cpp.git```
3. ```apt-get install libopenblas-dev``` 安装blas库，这里选用openblas。
4. 修改pkgconfig/openblas.pc或Makefile解决llama.cpp对openblas的依赖问题。最终要保证给编译期提供正确的LD Flags。
5. ```make LLAMA_OPENBLAS=1``` 完成编译。
6. ```./main -m /path/to/mistral-7b-instruct-v0.1.Q5_K_M.gguf --color -c 2048 --temp 0.8 --repeat_penalty 1.1 -n -1 -i -p "xx"```

64物理核打满，采样速率4000t/s，prompt处理速率80t/s，生成速率20t/s。

## Apple M1 Pro + macOS Monterey
1. 下载模型和llama.cpp repo。
2. 直接```make```就默认启用metal gpu加速和Accelerated Framework。
3. ```./main -m /path/to/mistral-7b-instruct-v0.1.Q5_K_M.gguf --color -c 2048 --temp 0.8 --repeat_penalty 1.1 -n -1 -i -p "xx"```

M1 GPU打到90%（剩下还有10%WindowServer在用），采样速率9000t/s，prompt处理速率60t/s，生成速率20t/s。

一边看视频（chrome GPU占用约20%），一边跑mistral也能有19.69t/s。

## Token Sampling
不同的sampling机制对文本生成有显著影响，本地LLM的可玩性来源很大程度上就是客制sampling。

Transformer模型根据前n个token，预测下一个token。每个token都有其next tokens的score，而next tokens的取值范围就是vocabulary（transformer架构最后有一个linear layer，将输出映射到vocabulary上，每个词都有对应的），这些score经过softmax将score数组转化成probability数组，根据probability挑选下一个token的过程就称为token sampling。
- Greedy采样： 如果每次都确定性地选择probability最高的那个token，单步决策，就是greedy sampler，优势是速度快，劣势是牺牲了模型的多样性，容易陷入重复循环和对训练数据的过拟合。llama.cpp里设置--top_p 0就相当于开greedy采样。
- Top-k采样： 从概率分布的前k个token里面进行随机采样。
- Top-p采样： 引入超参p，把token probability降序排序后，选取前面一部分，使这部分概率和为p，而不是固定选前k个token。

在使用LLM时，采样机制不同，效果也会大不相同。想要更高的确定性，更朴实无华的预测，则可以尝试greedy、低k的topk、低p的top-p采样。想要多样性和新颖性，则可尝试高k的top-k和高p的top-p。

除了top-k，top-p外，llama.cpp还实现了一些额外的采样机制：
- Min p filter：允许采样环节剔除低于阈值的低分候选。
- Tail free sampling：根据概率的二阶导数之和采样，即根据概率降速剔除尾部低概率tokens。
- Locally typical samplling：参数控制是否倾向局部语境内的典型的tokens。
- Mirostat sampling：[Mirostat算法](https://arxiv.org/abs/2007.14966)会调整top-k的k，避免陷入boredom trap（模式崩塌）和perplexity trap（不一致）。
- logit bias: 人为指定某个token的优先级，比如--logit-bias 29905-inf就把"\"token设为负无穷。
- temperature: 在对score向量（logits）做softmax前，把logits/temperature，则temp越高，softmax后概率分布的高低悬殊就会越接近，也就更有利于低概率tokens崭露头角。

## Prompting
让llm假装自己是一个javascript console，然后进行交互式的指令应答，甚至还能进行简单的算术计算，理解函数调用的返回值类型。当然，不能对正确性抱有期待。
![console](https://cmbbq.github.io/img/js_console.png)

用一大段prompt，让llm假装自己是一个爱说emoji，语言风格浮夸的音乐推荐bot，效果相当不错。
![bot](https://cmbbq.github.io/img/music_bot.png)



