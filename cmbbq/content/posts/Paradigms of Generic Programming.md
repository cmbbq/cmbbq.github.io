 
+++
title = "Paradigms of Generic Programming: Archetype, Ducktype, Subtype"
date = "2022-11-10"
tags = ["pl", "se"]
description = "本文总结泛型编程的三种范式：Archetype, Ducktype, Subtype。三者都以type结尾，一方面是因为这样比较帅，有规则感和逻辑上的建筑美，另一方面是因为系统语言编程本身就是在打造一个个类型，而泛型编程就是在打造一个个类型规约+遵循规约的类型。"
showFullContent = false
+++

## 泛型不等价于模板
什么是泛型编程？提起generic programming，很多人都会立刻想到template，但模板编程只是泛型编程的一种范式。最初发明"generic programming"这种说法的Alexander Stepanov反复强调过：
generic programming is not about how to use template。第一个版本的STL尽管名字里带template，实际上是基于scheme实现的。

## 要泛型，就必须有规约
规约的制定和对规约的遵循是泛型编程的基础。
任何泛型编程都需要一定规约，有规约才能让一种抽象适配多个具体实体。没有规约，那写具体实现的人都不知道遵循什么规范，实现什么接口，满足什么条件，泛型编程自然就无从谈起。

规约：在JavaScript里是prototype；在Swift里是protocol；在Rust里是trait；在C++模板编程里是concept或者SFINAE。

遵循规约：在JavaScript里是运行时将具体对象关联到某个原型对象；在Swift里是用类似继承的冒号让具体类型conforms to某些protocol；在Rust里是用impl SomeTrait for SomeStruct语法显式地为某个类型提供某个trait的实现；在C++里模板实参无需特殊语法声明遵循模板形参，但要心照不宣地遵循模板代码的要求。

## 三种规约，三种范式
根据规约不同，可命名泛型编程的三种范式：Archetype, Ducktype, Subtype。
- **Archetype**：以类型的类型(type-of-type)或者说协议(protocol)、接口(interface)为规约。
  - [Rust trait](https://doc.rust-lang.org/book/ch10-02-traits.html): "defines shared behavior"
  - [Carbon interface](https://github.com/carbon-language/carbon-lang/blob/trunk/docs/design/generics/terminology.md#interface): "defines an API that a given type can implement"
  - [Swift protocol](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/protocols/): "defines a blueprint of methods, properties, and other requirements"
  - [C++ type erasure idiom](https://davekilian.com/cpp-type-erasure.html): "captures the concept shared among all the concrete types"

- **Ducktype**：基于模板进行文本替换的结构化规约。
  - 模板本质上就是编译期ducktyping，不能提前独立进行类型和语法检查，只有实例化之后才能做类型和语法检查，能鸭子叫，编译通过，叫不出鸭子叫，编译报错。
  - C++20中模板结构化规约是concept, 此前则是SFINAE技巧，或者干脆不成文：要么因循旧例(iterable的T一定要有begin和end)，要么心照不宣（自己写的代码，只有自己懂，无需对外公开规约）。
  - 由于规约本身并非类型，无法用普通容器存储遵循规约的一系列具体类型，只能用类型推导的tuple-like容器。

- **Subtype**: 以基类为规约。
  - 子类系统是面向对象语言最普及的泛型编程范式，往往基于class hierarchy +『虚表存放函数指针』+『对象内置虚指针』或『callsite胖指针』实现。
  - 子类系统固然是简单自然的多态，但毕竟subtyping有一丢丢的性能开销，不是零成本抽象，而且难以非侵入式地让一个新接口适配已有代码。

本节命名的三种范式名称恰好都以type结尾，一方面是因为这样比较帅，另一方面是因为（对C++/Rust这种抽象层次的语言来说）编程本身就是在打造一个个类型，泛型编程就是在打造一个个类型规约+遵循规约的类型。Archetype和Subtype范式中，类型规约恰好就是一种抽象类型，所以这两种范式写起来更简单自然。Ducktype范式（C++模板）中，类型规约是结构化规约，可以是语言实体(concept)，也可以心照不宣，无论如何都不是类型。

| Paradigms | Archetype | Ducktype | Subtype |
| :----: | :----: | :----: | :----: |
| 具体语言实例 | Swift protocol, Carbon interface, Rust trait | C++ template(constrained or not), Rust generic | C++/Java/Python class hierarchy |
| 在哪里写泛型代码？ | 泛型函数 | 函数模板 | 子类方法 |
| 承载泛型代码的语言实体 | 普通函数，只不过是以规约类型为参数 | 模板文本 | 普通函数，不过函数指针被语言特性暗中与callsite指针绑定了 |
| 规约 | 定义一些类的方法或属性共性的协议类型 | 模板参数应符合的一组需求闭集，可以是成文约束Concept或SFINAE，也可以是不成文约束 | 面向对象语言继承图中的某个基类的虚函数 |
| 承载规约的语言实体 | 类，类型擦除之后的共性类，类型的类型，本质上仍然是类型 | 对文本替换的placeholder的结构化约束。即使约束了，我们也无法对placeholder做类型/语法检查 | 被基类列入必要功能列表的函数 |
| 遵循规约的语言实体 | 具体的类 | 用于模板实例化的模板参数 | 子类 |
| 何时做name lookup决议？| 早绑定，允许独立编译| 晚绑定，实例化时，不用的话就一直不绑定| 早绑定，允许独立编译|
| 何时做类型检查？| 独立编译时	| 实例化后	| 独立编译时 |
| 是否允许支持动态绑定？| 是 |否 |是 |
| 如何将已有泛型接口在新的类型上进行扩展？|	允许基于已有类型创造一个数据表示兼容的新类型，只不过规约变了，允许它实现某个新的规约，或提供与原类型的已实现的某个规约提供不同的实现。| 为新偏特化场景写新的函数模板即可扩展，可用Concept或SFINAE修订重载决议规则。也可以在函数模板里用某个约定好的函数名、成员变量名、关联类型作为客制点，新的类型只需实现这些客制点。| 继承，子类数据表示可能会变 | 

## Archetype范式
Archetype在Rust中是语言的核心机制——trait，要写好Rust的泛型代码，就要习惯于基于archetype编程。相比与DuckType、SubType，Archetype范式天然有一些优势。

### Generic code is just normal code
和模板相比，"Generic code is just normal code"是Archetype范式最大优势，也是Rust语言相比C++的巨大优势。让非专家也能写出高度抽象，同时还零成本，具有高度可复用性的泛型代码。

C++基于模板文本替换的泛型代码和普通类型的具体实现代码，在写法上有一定区别，编译链接也有区别。
这是因为C++中基于concept模板，其实是没办法提前编译、提前语法检查，只能在实例化之后再做语法检查。所以是一种难度相当高的编程范式，能用模板写C++库的一般是业界专家，普通人很难掌握，也没有足够动机去掌握。

Rust基于trait的泛型代码则和普通类型的具体实现代码，在写法上基本没区别，编译链接方式也一样。
这是因为Rust中的trait是一种archetype，type-of-type，是规定一组类型应该具备什么接口的元类型，无论它多么抽象多么特殊，究其本质仍是一种类型。只要是类型，就可以单独提前编译，就可以被提前语法检查。

### Adapting Erases Interoperability 
和继承相比，"Adapting rather than extending a type"是Archetype范式的优势之一。
Subtype范式在实践中不能保证所有类型继承自同一个基类，比如说对第三方的代码没有控制权，或者说这个类型不是个class，而是int, float这种内置类型。
Archetype范式中不仅可以为自己的类型提供多种Archetype adaption，或使自己的代码遵循第三方Archetype，还可以为第三方类型提供自己的Archetype实现——前两点还好，这最后一点是Subtype范式做不到的，只能加个丑陋的wrapper，不仅工作量特别大，而且容易出错。

有人会问，允许修改已有的类型是不是比较危险？这是一种误解。继承是修改，因此是危险的。Adapt（或者说override，newtype）其实不是修改，而是新增。
继承改变了类型的数据表示，Adapt机制则不改变类型的数据表示，只为其新增接口——换一种说法，override Archetype for T机制实际上是为已有类型T新建了一个遵循规约Archetype的入口。

### Archetypes in C++
有些人会argue，C++无所不能，的确C++也可以实现archetype范式，比如std::function，以及其他类型擦除。但是基于现有语法写出来的泛型代码和普通代码之间还是没那么像。One has to drastically change the programming style in order to "go generic". 实现archetype范式在C++中相对困难。但即使如此，archetype范式的固有优势还是让某些标准或准标准选择了它，比如std::function, std::any, boost::any_range。