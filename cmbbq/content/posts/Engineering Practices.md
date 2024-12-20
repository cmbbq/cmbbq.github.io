+++
title = "Idiomatic Practices in C++ Systems Engineering"
date = "2024-12-03"
tags = ["sys", "se"]
description = "本文总结当下我认为比较好的C++系统工程范式。"
showFullContent = false
+++

## 基于真实需求，创造新颖的计算形态。
- 若一个系统不存在新颖性，则不必为审美或宗教原因重复造轮。
- 应认识到，绝大多数系统在商业上几乎没有价值，本就不应该存在。自然不应该向本就不该存在的系统投入稀缺的工程人力。

## 用最小技术集构建当前问题的直接解。
- 如果已有技术修修补补后只能勉强解决问题，存在不可克服的缺陷，则考虑研发新系统作为直接解。
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
- 兼顾人体工学与性能工程，在不同场合做不同的取舍：
  - nonexpert-transparent：让大多数代码逻辑足够简单易懂、简洁直接，以至于任何背景的研发都能不借助文档、注释轻松读懂代码。
  - expert-friendly：与此同时，代码库中性能攸关部分，必须能允许专家施加任意强度的优化。任何人体工学驱动的抽象都不能影响到性能工程的自由度。

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
  - 问题是ctor没有返回值，这其实也是最初C++需要设计exception的原因。作为workaround，可以选择提供一个static make/create函数，返回expected<T,E>。
  - 顺便避免copy ctor。大多数类不需要拷贝构造，少数需要的情况应显式实现一个copy/clone函数。
  - C++没有Rust的?语法糖，但可以自己实现一个try宏[^5]。
- 非对外接口，非性能攸关，错误处理并不重要，且只需向上传播的少数场景，宜用exception——且最好用gcc14.2+[^3]（对exception性能大有提升）。
  - 这样的场景并非完全不存在，典型的例子是json parsing，你并不需要对每个位置、各种情形的json错误做出不同的应对，大多数时候只需要打印出来哪一行json格式不对就行了。
  - 此外，各种一次性脚本、临时任务、或简单应用等速朽场景，甚至连exception都不一定要用，自然也不需要general-purpose library或data-center application级别的error handling和接口设计，能用就行。

## 设计之初就将系统中的错误分类，明确责任边界
- 错误可分为：用户错误（invalid_input）、可容忍系统错误（glitch）、不可容忍系统错误（fatal）、编程错误（bug）。
- 用户输入错误输入是常态，应将其视为正常路径处理，不应使用exception，不用做任何补救和恢复，尽快返回错误消息让用户知悉即可。
- 系统错误是某个下游组件或某个系统调用失败导致的错误，这种错误应被视为系统故障，其中不可容忍的严重故障应与普通故障区分开——以便给它们单独的日志等级，引入某种告警和人工介入机制。
- 编程错误是理想代码中本不应该出现的断言失败，有些是precondition检查失败，有些是postcondition检查失败，这些都可以归类为bugs，发现后需进行修复。

## 慎重对待需要就地处理的局部错误。
- 错误是分层的，处理错误的context也是分层的，有些错误只能在它的上一层级得到妥善处理，一旦向上层传播，就会丢失正确处理它的context，因此有必要从接口设计层面慎重对待这类错误。
- 如果整个项目的std::expected<T,E>中的E都是一个全局错误类型，编程时可以非常轻松地进行error propagation。但这也意味着有些不该抛给上层的错误也容易被调用者抛上去。
- 应赋予需要就地处理的局部错误一个独占的类型[^1]，并用嵌套的expected表示返回值类型，如``expectd<expected<T, SpecialError>, Error>``，迫使调用者对SpecialError做单独处理。

## 用FTADLE实现多态。
- 我们当然不会用笨重的继承+动态绑定（subtype），也少用丑陋的模板+concept（ducktype）。
- 相比而言，FTADLE是更加简洁、灵活、美观、bug-free且易维护的定制化方案，巧妙利用了C++的一个冷门语言特性（[ADL](https://en.cppreference.com/w/cpp/language/adl)），实现了一种优雅的archetype多态。[^2]

## 让自定义类型尽可能平凡（trivial）
- 能安全的放进各种容器，即copy assignable + copy constructible。
- 按bits复制（比如std::memcpy）可正确完成对象的复制，即trivially copyable。
- 少数确实不宜设计为copyable的类型，保证trivally move assignable/constructible。
- 可无异常默认构造，即nothrow default constructible。
## 用强类型表达约束
- 强类型作为参数，可以消除一些函数的隐式的前置条件（implicit preconditions）。
  - 比如std::string的back()函数返回char&，隐含了该string非空的前置条件。[^4]
  - 很多参数应该属于它的类型，但并不是所有这个类型的值都是合法参数，比如path用std::string来存很合理，但不是所有string都是合法path。
  - 用non_empty_string作为string类型，就可以消除非空这一前置条件。
  - 类似地，可以用更特殊的类型将表达各种约束。
- 在头文件中定义好约束更强更安全的基础类型。
  - 比如禁止和有符号数加减转换的8bits无符号数：``using u8 = type_safe::integer<uint8_t>;``
  - 比如禁止从0/1转换的布尔类型：``using boolean = type_safe::boolean;``
  - 比如禁止用operator==()的浮点数类型：``using f32 = type_safe::floating_point<float>;``


[^1]: [Error Handling](https://cmbbq.github.io/posts/error-handling)
[^2]: [Paradigms of Generic Programming: Archetype, Ducktype, Subtype](https://cmbbq.github.io/posts/paradigms-of-generic-programming/)
[^3]: [C++ exception performance three years later](https://databasearchitects.blogspot.com/2024/12/c-exception-performance-three-years.html)
[^4]: [Prevent precondition errors with the C++ type system](https://www.foonathan.net/2016/09/error-handling-types/)
[^5]: 可以考虑基于[StatementExprs](https://gcc.gnu.org/onlinedocs/gcc/Statement-Exprs.html)实现try宏，模拟Rust的?操作符，clang也有类似机制。

