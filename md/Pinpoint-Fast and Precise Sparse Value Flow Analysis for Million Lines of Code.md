# Pinpoint: Fast and Precise Sparse Value Flow Analysis for Million Lines of Code

## Abstract

When dealing with millions of lines of code, we still cannot have the cake and eat it: sparse value-flow analysis is powerful in checking source-sink problems, but existing work cannot escape from the "pointer trap" - a precise points-to analysis limits its scalability and an imprecise one seriously undermines its precision.We present Pinpoint, a holistic approach that decomposes the cost of high-precision points-to analysis by precisely discovering local data dependence and delaying the expensive inter-procedural analysis through memorization. Such memorization enables the on-demand slicing of only the necessary inter-procedural data dependence and path feasibility queries, which are then solved by a costly SMT solver. Experiments show that Pinpoint can check programs such as MySQL (around 2 million lines of code) within 1.5 hours. The overall false positive rate is also very low (14.3% - 23.6%). Pinpoint has discovered over forty real bugs in mature and extensively checked open source systems. And the implementation of Pinpoint and all experimental results are freely available.

当处理数百万行代码,我们仍不能有蛋糕和食用方法:稀疏的价值流分析是强大的检查源库问题,但现有的工作不能逃离“指针陷阱”——一个精确的指向分析限制了其可伸缩性和一个不精确的严重削弱了其精度。我们提出了精确定位，一种整体的方法，通过精确地发现局部数据依赖，并通过记忆延迟昂贵的程序间分析，将高精度点的成本分解为分析。这种记忆只允许按需切片必要的过程间数据依赖和路径可行性查询，然后由昂贵的SMT求解器解决。实验表明，查明可以在1.5小时内检查MySQL(大约200万行代码)等程序。总体假阳性率也很低(14.3% - 23.6%)。查明已经在成熟的和广泛检查的开源系统中发现了超过40个真正的bug。并且实现了精确定位，所有的实验结果都是免费提供的。

## 1 Introduction

The analysis of value flows underpins many recent techniques in statically finding bugs such as null pointer deference [3, 29, 30, 53], memory leak [9, 45, 47, 52], use-afterfree and double-free [9, 16, 21]. Unfortunately, despite this tremendous progress, we still observe the difficulty of applying value-flow analysis at industrial scale - finding bugs hidden behind sophisticated pointer operations, deep calling contexts, and complex path conditions with low false positive rates, while sieving through millions of lines of code in just a few hours.

- [3] D. Babic and A. Hu. 2008. Calysto: Scalable and Precise Extended Static Checking. In 2008 ACM/IEEE 30th International Conference on Software Engineering (ICSE 2008). IEEE, 211-220.
- [29] David Hovemeyer and William Pugh. 2007. Finding more null pointer bugs, but not too many. In Proceedings of the 7th ACM SIGPLANSIGSOFT workshop on Program analysis for software tools and engineering. ACM, 9-14.
- [30] David Hovemeyer, Jaime Spacco, and William Pugh. 2005. Evaluating and tuning a static analysis to find null pointer bugs. In ACM SIGSOFT Software Engineering Notes, Vol. 31. ACM, 13-19.
- [53] Yichen Xie and Alex Aiken. 2005. Scalable Error Detection Using Boolean Satisfiability. In Proceedings of the 32nd ACM SIGPLANSIGACT Symposium on Principles of Programming Languages (POPL ’05). ACM, 351-363.
- [9] Sigmund Cherem, Lonnie Princehouse, and Radu Rugina. 2007. Practical Memory Leak Detection Using Guarded Value-flow Analysis. In Proceedings of the 28th ACM SIGPLAN Conference on Programming Language Design and Implementation (PLDI ’07). ACM, 480-491.
- [45] Yulei Sui and Jingling Xue. 2016. SVF: Interprocedural static value-flow analysis in LLVM. In Proceedings of the 25th International Conference on Compiler Construction. ACM, 265-266.
- [47] Y. Sui, D. Ye, and J. Xue. 2014. Detecting Memory Leaks Statically with Full-Sparse Value-Flow Analysis. IEEE Transactions on Software Engineering 40, 2 (2014), 107-122.
- [52] Yichen Xie and Alex Aiken. 2005. Context-and path-sensitive memory leak detection. In ACM SIGSOFT Software Engineering Notes, Vol. 30. ACM, 115-125.
- [16] David Dewey, Bradley Reaves, and Patrick Traynor. 2015. Uncovering Use-After-Free Conditions in Compiled Code. In Availability, Reliability and Security (ARES), 2015 10th International Conference on. IEEE, 90-99.
- [21] Josselin Feist, Laurent Mounier, and Marie-Laure Potet. 2014. Statically detecting use after free on binary code. Journal of Computer Virology and Hacking Techniques 10, 3 (2014), 211-217.

对值流的分析是静态查找bug的许多最新技术的基础，比如空指针遵从[3,29,30,53]、内存泄漏[9,45,47,52]、后用和双用[9,16,21]。不幸的是,尽管如此巨大的进步,我们还观察到工业规模应用价值流分析的难度——找到错误隐藏在复杂的指针操作,深调用上下文,和复杂道路条件较低的误判率,同时筛选通过数百万行代码在几个小时。

Techniques following the design of conventional data-flow analysis and symbolic execution, such as the IFDS framework [39], Saturn [53], and Calysto [3], propagate dataflow facts to all program points following control-flow paths. These “dense” analyses are known to have performance problems [9, 38, 47, 49]. For example, a recent report [3] observes that it takes 6 to 11 hours for Saturn [53] and Calysto [3] to check only one property (null pointer dereference, a typical source-sink problem) for programs of 685 KLoC.

遵循传统数据流分析和符号执行设计的技术，例如IFDS框架[39]、Saturn[53]和Calysto[3]，将数据流事实传播到遵循控制流路径的所有程序点。这些“密集”分析被认为存在性能问题[9,38,47,49]。例如，最近的一份报告[3]观察到，土星[53]和花萼[3]只需要6到11个小时来检查一个属性(空指针反引用，一个典型的源-接收器问题)对于685 KLoC的程序。

Sparse value-flow analysis (SVFA) [9, 35, 43, 45, 47] mitigates this performance problem by tracking the flow of values via data dependence on sparse value-flow graphs (SVFG), thus, eliminating the unnecessary value propagation. These techniques are referred to as the “layered” approaches due to the need of discovering data dependence as a priori through an independent points-to analysis. However, since a highly precise points-to analysis is difficult to scale to millions of lines of code [28], these “layered” SVFA techniques often give up the flow- or the context-sensitivity in the points-to analysis and avoid using SMT solvers to determine path-feasibility, such as in the cases of Fastcheck [9] and Saber [47]. Choosing a scalable but imprecise points-to analysis blows up the SVFG with false edges, overloads SMT solvers, and generates many false warnings, which we refer to as the “pointer trap”. In practice, we observe that developers have very low tolerance to such compromises because forsaking any of the following goals - scalability, precision, the capability of finding bugs hidden behind deep calling contexts and intensive pointer operations - creates major obstacles of adoption.

稀疏值流分析(SVFA)[9, 35, 43, 45, 47]通过数据依赖于稀疏值流图(SVFG)来跟踪值的流动，从而消除了不必要的值传播，从而缓解了这个性能问题。这些技术被称为“分层”方法，因为需要通过独立的点到分析来先验地发现数据依赖性。然而,由于一个高度精确的指向分析很难扩展到数百万行代码[28],这些“分层”SVFA技术往往放弃指向的流或上下文敏感性分析和解决避免使用SMT确定path-feasibility,等情况下的Fastcheck军刀[47]和[9]。选择一个可伸缩但不精确的点- - -分析用假边摧毁SVFG，重载SMT求解器，并生成许多假警告，我们将其称为“指针陷阱”。在实践中，我们观察到开发人员对这种折衷的容忍度很低，因为放弃以下任何一个目标——可伸缩性、精确性、发现隐藏在深层调用上下文和密集指针操作背后的bug的能力——会造成采用的主要障碍。

In this work, we make no claims of breakthroughs to the innate scalability limitations of points-to analysis and solving path conditions using SMT solvers. However, we note that the conventional “layered” approaches can significantly exacerbate the impact of these limitations on the perceived performance of SVFA, for which we are able to address. Our key insight is that an independent points-to analysis is unaware of the high-level properties being checked and, thus, causes a great deal of redundancy in computing pointer relations.

在这项工作中，我们没有声称突破了点的固有可伸缩性限制——使用SMT求解器分析和解决路径条件。然而，我们注意到传统的“分层”方法会极大地加剧这些限制对SVFA感知性能的影响，而我们能够解决这些问题。我们的主要观点是，独立的points-to分析不知道要检查的高级属性，因此会在计算指针关系中造成大量的冗余。