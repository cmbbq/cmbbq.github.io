+++
title = "Idiomatic Practices in C++ Systems Engineering"
date = "2024-12-03"
tags = ["sys", "se"]
description = "本文总结当下我认为比较好的C++系统工程范式。"
showFullContent = false
+++

## 用最小技术集构建当前问题的直接解。
- 如果已有技术修修补补后勉强解决问题，但又存在根本缺陷，则考虑研发新系统作为直接解。
- 不为未来的不确定需求破坏当前方案的简洁性。
- 管理复杂度，提升可理解性和可维护性。
  - 核心代码fit in人脑短期记忆的解决方案是技术资产，反之则是技术债。
  - 若核心代码复杂度高到无法fit in单人短期记忆，则将其合理切分成多个模块，交由多人维护。
  - 避免引入过多的第三方库，规避C++ dependency hell。
  
## 避免解释型注释，让代码自解释。
- 如果某一行代码做了什么需要注释，那么说明它还可以进行更好的重构。
  - 这种注释就像todo标志，表明有空就应该把它重构一下，提升代码质量的同时顺便去除注释。
  - 剔除这类注释，可以避免日后迭代中，注释逐渐和代码对不上，对读者产生灾难性的误导。
- 尽量阐释为何这么做(WHY)，而不注解做了什么(WHAT)。
  - 比较复杂的场景，可以用大段文字整体介绍设计思路。
  - 这段文字可以放在代码最前面醒目位置，方便随着版本更新也进行相应更新。不必写额外的文档，或Readme，因为外置的文档和代码之间的映射也是脆弱、难以长期维护的。

## 使每个编译单元内自带文档、单测和构建指令
- 编译单元(.cc和它包含的头文件)承载了某个功能的实现，但对这个功能的测试、文档描述、构建脚本却往往放在其他文件里，有时会在很奇怪的位置，甚至混杂在其他复杂文件中，对于新接手项目的人来说了解这个编译单元的全貌就变得很麻烦。
- 所以不妨直接把单测代码写在每个.cc文件最下面，把单测的构建信息以注解形式写在.cc文件最上方，把文档以大段注释的方式写在.cc/.h的醒目位置。
- 如无特殊需求，构建工具不妨就用[emake](https://github.com/skywind3000/emake)。

## 非性能攸关场景，尽可能拆分出更多函数，让每个函数只做一件事。
- 如果一个复杂函数能够拆解成多个函数，则将其拆解。
- 如果多个函数共享一些变量，则重构成类。

## 尽可能避免使用exception，而是使用result monad。
- 避免异常引入不必要的性能开销。
- 唯有禁用异常，才能convey fallibility through APIs。
- std::expected<T,E>或best::result<T,E>都是不错的选择。

## 慎重对待需要就地处理的局部错误。
- 错误是分层的，处理错误的context也是分层的，有些错误只能在它的上一层级得到妥善处理，一旦向上层传播，就会丢失正确处理它的context，因此有必要从接口设计层面慎重对待这类错误。
- 如果整个项目的std::expected<T,E>中的E都是一个全局错误类型，编程时可以非常轻松地进行error propagation。但这也意味着有些不该抛给上层的错误也容易被调用者抛上去。
- 应赋予需要就地处理的局部错误一个独占的类型[^1]，并用嵌套的expected表示返回值类型，如``expectd<expected<T, SpecialError>, Error>``，迫使调用者对SpecialError做单独处理。

## 用FTADLE实现多态。
- 我们当然不会用笨重的继承+动态绑定（subtype），也少用丑陋的模板+concept（ducktype）。
- 相比而言，FTADLE是更加简洁、灵活、美观、bug-free且易维护的定制化方案，巧妙利用了C++的一个冷门语言特性（[ADL](https://en.cppreference.com/w/cpp/language/adl)），实现了一种优雅的archetype多态。[^2]

[^1]: [Error Handling](https://cmbbq.github.io/posts/error-handling)
[^2]: [Paradigms of Generic Programming: Archetype, Ducktype, Subtype](https://cmbbq.github.io/posts/paradigms-of-generic-programming/)

