+++
title = "String Lookups Could Reduce to Parsing"
date = "2023-05-14"
tags = ["sys"]
description = "字符串查找和字符串解析，本质都用尽可能紧凑的结构和高效的算法，从字符流中抽取状态。因此龙书中的NFA转DFA算法可以派上用场。"
showFullContent = false
+++

标题即结论：字符串查找问题可归约为解析问题。

这个结论源于近期一个有趣的观察：用ragel写高性能的ascii protocol parser，本质上是利用[nfa](https://en.wikipedia.org/wiki/Nondeterministic_finite_automata)转[dfa](https://en.wikipedia.org/wiki/Deterministic_finite_automaton)提升性能，这和[toplingdb](https://github.com/topling/toplingdb)中将同一层各个sst对应的trie（本质是dfa）合并成一个dfa(大量dfa->nfa->1dfa)的思路是同构的。

上述同构隐隐蕴含着一个reduction：字符串查找和字符串解析，本质都用尽可能紧凑的结构和高效的算法从字符流中抽取状态，lookups可以视作一类特殊（且模式相当规则）的parsing。LSM key lookup更是比较特殊的海量索引的key range无overlap的场景，存在大量可以轻松合并的DFA。因此龙书中的大量DFA转NFA转DFA算法可以派上用场。

KV Store的in-memory key index，可以是红黑树，可以是skiplist，可以是hashmap，可以说patricia trie(一种[radix tree](https://en.wikipedia.org/wiki/Radix_tree)变种)，也可以是toplingdb中的NestLoudsTrie。这种结构我们同样可以在路由表实现中看到。字符串索引，归根到底是字符串lookup结构。正如路由表实现可以通过把所有routes合并到一个DFA里（很多routes都包含regex），kv数据库也把Trie这种特殊的dfa（Trie的状态转移图是树，树是一种无向图，多了个任意两结点只由一个边连接的约束）做多索引合并，每个索引对应的key range还不重叠（LSM特性），因此合并速度非常快，合并后的DFA表示起来也简单、紧凑，详见[自动机算法在数据库索引中的应用](https://zhuanlan.zhihu.com/p/628057993)，我在作者的文章下面追问了一下DFA合并的触发条件和DFA合并开销，作者的答复是compaction/flush时触发，在整个lsm更新过程中占比很小，也不涉及多线程，无需考虑线程安全。





