+++
title = "Error Handling"
date = "2024-04-09"
tags = ["se"]
description = "本文讨论现代C++的错误处理问题。"
showFullContent = false
+++
本文讨论现代C++的错误处理问题，结论如下：
1. 宜用``Result Monad``取代C范式和C++异常，在接口层面清晰定义表明可出错性，和错误清单。
2. 每个模块应有独立的Error类型，不宜用全局统一的Error类型或其subtype/variant，规避过于丝滑的错误传播导致的舒适陷阱。
3. 上述模块级的Error类型一般就是一个``enum class``，足够紧凑、安全。必需在Error中留存状态的场景再用``std::variant``+``std::visit``。
4. 明确需特殊处理的错误宜独占一个类型，不宜和常规错误一起并列在模块级Error的enum class中。

# Exception Sucks in Every Possible Way
C++的异常具有多个令人绝望的特质：
- 违反C++的基本设计准则——``zero cost abstraction``。
- 鼓励隐藏``control flow``。
- 不鼓励在接口中给出错误定义。
- 对异常类型没有任何约束。
- 不提供有界的空间时间开销承诺。``backtrace``耗时多久完全不可预计。
- 导致编译后的代码膨胀。
- 不兼容``C ABI``。
- ``throw``需要动态内存分配，``catch``甚至需要``RTTI``。嵌入式环境下不会这么奢侈。

总之C++异常机制是一种自然演化而成的历史遗迹，而非合乎理性的程序语言设计。以今天的标准，连写成提案的机会都不会有就会在mailing list被C++语言律师冷嘲热讽而消失。

哪怕现在有很多轻量化甚至零成本抽象的新型exception提案（比如P1095R0[^1]和P0709R4[^2]），一时半会儿也不会有改观，固有的exception已经根植于历史系统中，包含标准库在内的庞大基础库都难以承受范式迁移的代价。合理对待异常的态度就是弃之不用。

# Convey Fallibility in API 
C风格error code虽然简单，但需要使用者有强大的自控能力，对于现代编程来说还是太过危险、简陋，语言层面本身无法区分参数中哪个是输入，哪个是输出，仰赖约定俗成的命名习惯或注释，缺乏可读性，编程时的心智负担太重。错误传播则缺乏类型安全，往往直接传引用和指针，引发难以追踪的内存安全、线程安全问题。不做任何错误传播，就地解决错误的话，又会导致代码繁琐冗余，总有一天会懒得做错误处理。

如何兼具安全的错误传播、零成本抽象、接口中传达清晰的错误清单？Golang也没做到（几乎和C一样简陋，给了官方的error类型，然而并非泛型或Monad，只是一个返回Error()字符串的接口）。Java有点作弊（封闭且完整的JVM+标准库+Javadoc注释生态圈）。Rust则轻装简行，为我们展示了``Result<T,E>``的巨大成功——后续我们将其称为Result范式，用函数式编程语境下的``Result Monad``指代这类结构。

抛开性能、C-兼容性等问题不谈，纯粹从软件系统设计的角度讲，``Result Monad``也能更好地在API层面清晰无误地将可能出现的错误良好定义。而exception则散落在代码实现中，单看接口根本不知道throw了什么，隐藏了哪些坑，如何处理这些坑——这在软件互操作性上是灾难性的。

如今Rust-like Result范式的[std::expected](https://en.cppreference.com/w/cpp/utility/expected)已经进入标准库，自gcc12/clang16起已经可以使用了，受限于旧版编译器的场景则可以用第三方实现。

# Keep Local Errors in Their Own Types
基于``Result<T,E>``或``expected<T,E>``进行错误处理时，有一个简单方便的做法是将E设置成一个巨大的全局enum，包含所有可能的error code。这样就可以在整个程序中自由地用``?``操作符或``and_then/or_else``传播error，达到类似``throw``+``catch``异常的效果。

这么做的确可以降低编程时的心智负担，但同时也注入了滥用错误传播的风险——当编码者可以轻易甩锅时，往往就会甩锅。通常一个独立且内聚的模块会对自身内部细节有更多了解，有些问题还是就地处理更妥当，把失败的细节暴露给相对外行的调用者，是在鼓励制造不恰当的耦合。

因此每个系统模块对外应暴露最小化的错误集，将内部可处理的错误在内部消化，并将所有``fatal error``就地解决——往往是打日志、做些修复（有状态系统）、退出程序。为了避免滥用错误传播，对外暴露的这个错误集还应该有自己的enum类型（C++语境下一般就是``enum class``，如有必要，可尝试用``std::variant``+``std::visit``模拟Rust的``Enums`` + ``pattern matching``，见[^5]），而不宜采用全局统一的某种enum的subtype或variant——迫使直接调用者优先对错误进行处理而不是甩锅给间接调用者，在甩锅非常合理的场景下，必要的显式类型转换或``transform_error``/``map_error``，也让甩锅成为一个有意识的system2编程决策，而非system1本能行为。

此外，有一些特殊错误的处理逻辑与众不同，不宜和其他错误共享一个``enum class``，而应赋予这单个error一个独占的类型。这种情况下，可以用嵌套的expected表示返回值类型，如``expectd<expected<T, SpecialError>, Error>``，迫使调用者对SpecialError做单独处理。一个典型的例子是``sled``[^4]中``compare_and_swap``，返回类型特地包裹了两层，外层是``sled::Error``，内层是特殊的``CompareAndSwapError``（CAS失败并非异常，而是常态，对它的处理应视为控制流，而非异常处理），在做这种设计后，``sled``用户误用这个函数的几率就大大降低了，第一次用``?``操作符仅将外层的通用Error传播了出去，不至于把必需特殊处理的CAS error也一并甩出去。

```Rust
fn compare_and_swap(
  &mut self,
  key: Key,
  old_value: Value,
  new_value: Value
) -> Result<Result<(), CompareAndSwapError>, sled::Error>

// we can actually use try `?` now
let cas_result = sled.compare_and_swap(
  "dogs",
  "pickles",
  "catfood"
)?;

if let Err(cas_error) = cas_result {
    // handle expected issue
}
```

另一个例子是需要rollback的atomic batch set操作，一旦失败就会rollback已执行的部分set操作，避免不一致。但这个batch set操作一旦rollback失败，则陷入无法自动恢复，必需人工干预的状态。如果把rollback error当做KvErrorCode中的一种，只返回``std::expected<void, KvErrorCode>``，是无法迫使上游用户区分对待rollback失败的——调用者要么把error统一向上传播，要么就简单打个日志，而我们希望调用者意识到数据不一致的灾难已经发生，至少得打个fatal日志。

```C++
std::expected<std::expected<void, KvErrorCode>, RollbackFatalError> BatchSet(const vec<std::pair<str, str>> &kvpairs);
```

总之，在定义类似``expected<expected<T,E1>,E2>``的``result monads``时，外层error类型``E2``永远要比内层error类型``E1``更加fatal，更加erroneous。在避免了灾难性的E2后，才有资格判断是否出现E1。

[^1]: [P1095R0: Zero overhead deterministic failure: A unified mechanism for C and C++](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p1095r0.pdf) 
[^2]: [P0709R4: Zero-overhead deterministic exceptions: Throwing values](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p0709r4.pdf) 
[^4]: [Error Handling in a Correctness-Critical Rust Project](https://sled.rs/errors) 
[^5]: [Rust enums in Modern C++ – Match Pattern](https://thatonegamedev.com/cpp/rust-enums-in-modern-cpp-match-pattern/)