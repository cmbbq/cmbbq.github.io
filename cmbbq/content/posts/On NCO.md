+++
title = "On NCO"
date = "2022-10-12"
tags = ["ai"]
description = "非凸优化(non-convex optimization)，more like art"
showFullContent = false
+++

凸优化收敛时间一般是polynomial的，线性规划和最小二乘就是凸优化的特例。

非凸优化non-convex optimization是一种至少np-hard的问题，不存在通用解法。想要确定问题是否有解，局部最优是否全局最优，或目标函数是否有界都会随着变量和约束数目指数爆炸blow up，局部优化手段对算法参数敏感，又高度依赖initial guess，这使局部非凸优化more-art-than-technology，相比而言线性规划是毫无art可言的。

深度神经网络作为通用函数拟合器，最重要的作用是拟合非凸函数，因为复杂问题一般不可以用凸函数拟合。ChatGPT这类生成式模型就是对target和input间互信息的非凸优化。怎么训好模型，目前依然是一个art。

随机梯度下降stochastic gradient descent(SGD)被证明可以收敛于凸函数、可微和利普希茨连续函数，但还不能确定在非凸函数上的效果，SGD收敛缓慢，还不一定达到局部最优，更不一定达到全局最优。如果选择一个足够靠近全局最优的点，或许可以用SGD收敛到全局最优，但这一方面耗时间，另一方面只适用于特殊场景。对于深度神经网络来说，一旦陷入错误的局部最优，就要用不同的初始化配置或加入额外的梯度更新噪音。如果遇到鞍点，则需找到海森矩阵或计算下降方向。如果陷入低梯度区域，则需batchnorm，或使用relu做激活函数。如果因高曲率而使得steps过大，则应使用adaptive step size或限制梯度step尺度。此外，如果超参有问题，还需要用各种超参优化的方法。总之，目前深度学习的NCO还是处于art的阶段。