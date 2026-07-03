+++
date = '2026-07-03T00:00:00+08:00'
draft = false
categories = ["Static Analysis"]
tags = ["tai-e", "assignment"]
title = 'Tai-e Assignment 1: 活跃变量分析与迭代求解器'
+++

## 1 实验内容

- 为 Java 实现一个活跃变量分析（Live Variable Analysis）。
- 实现一个通用的迭代求解器（Iterative Solver），用于求解数据流分析问题，也就是本次作业中的活跃变量分析。

## 2 实验概览

这次的实验是“函数填空”的类型，实验给出基础框架，我们实现关键函数。

那了解这种项目最好的方式就是通过函数调用链一个个函数往上看，看看我们所实现的函数是怎么样被框架调用的。

第一个小实验是实现活跃变量分析，具体来说是要实现 LiveVariableAnalysis 中的如下 API：

- SetFact newBoundaryFact(CFG)
- SetFact newInitialFact()
- void meetInto(SetFact,SetFact)
- boolean transferNode(Stmt,SetFact,SetFact)

我们可以在IDEA中通过将光标移动到函数名上再按 Alt + F7 快捷键来查看这个函数在哪些地方被调用了。

不过上面这几个函数并没有找到被调用的地方，那我们可以大胆猜测这些函数是为接下来要实现的迭代求解器服务的。

第二个小实验是要实现两个方法：

- Solver.initializeBackward(CFG,DataflowResult)
- IterativeSolver.doSolveBackward(CFG,DataflowResult)

首先看第一个方法所涉及到的函数调用链，自上而下依次是：

```java
DataflowResult<Node, Fact> analyze(IR ir);

DataflowResult<Node, Fact> solve(CFG<Node> cfg);

DataflowResult<Node, Fact> initialize(CFG<Node> cfg);
```

首先看第一个函数：

```java
public DataflowResult<Node, Fact> analyze(IR ir) {
    CFG<Node> cfg = ir.getResult(CFGBuilder.ID);
    return solver.solve(cfg);
}
```

把IR转换成CFG，然后传给solver进行求解。

```java
public DataflowResult<Node, Fact> solve(CFG<Node> cfg) {
    DataflowResult<Node, Fact> result = initialize(cfg);
    doSolve(cfg, result);
    return result;
}
```
根据CFG初始化DataflowResult
- DataflowResult记录了CFG中所有Node的入口和出口的Fact。
- Fact是数据流分析在每个程序点所要传播和计算的那条信息（比如活跃变量分析中，Fact 就是"当前语句处哪些变量是活跃的"）

doSolve函数实际上调用的就是IterativeSolver.doSolveBackward(CFG,DataflowResult)，可想而知我们要在这个函数中实现我们所有的分析逻辑。

## 3 实验过程重点

实验本身不难，基本上跟着文档把算法翻译一遍就行，虽说还是有一些API即便我看过注释也不太清楚是怎么用的，但是上手把玩两下基本上也搞明白了。

在做实验的过程中我发现我对于算法的有些地方的理解是错的，比如，假设有三个节点A、B、C，并且有A->B，C->B，当我尝试更新B的in fact的时候，我误以为要将A和C的out fact一起更新掉，但其实是不用的。

## 4 杂项

## 4.1 调试

整个实验中，花时间最久的就是搞清楚该如何调试。

在调试的时候我尝试用print打印一些信息，但是结果发现日志全是乱的，这是因为分析的对象是一个java类，里面会有两个以上的方法：

- init() — 构造方法
- func(...) — 业务方法

AbstractDataflowAnalysis extends MethodAnalysis，框架会对被测程序的每个方法分别构建 CFG 并调用 solver.solve(cfg)，所以
doSolveBackward 执行多次——一次一个方法。

这种情况下想要调试还是只能用设置断点这种方法。
