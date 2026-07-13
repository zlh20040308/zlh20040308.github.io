+++
date = '2026-07-12T19:03:41+08:00'
draft = false
categories = ["Static Analysis"]
tags = ["tai-e", "assignment"]
title = 'Tai-e Assignment 3: 死代码检测'
+++

## 1 实验内容

- 为 Java 实现一个死代码（dead code）检测算法。

## 2 实验概览

在这次的实验作业中，我们将会通过组合前两次作业中实现的分析方法：活跃变量分析和常量传播，来实现一个 Java 的死代码检测算法。

在本次作业中，我们只关注两种死代码：不可达代码（unreachable code）和无用赋值（dead assignment）。

### 2.1 不可达代码

一个程序中永远不可能被执行的代码被称为不可达代码。我们考虑两种不可达代码：控制流不可达代码（control-flow unreachable code）和分支不可达代码（unreachable branch）。

#### 控制流不可达代码

在一个方法中，如果不存在从程序入口到达某一段代码的控制流路径，那么这一段代码就是控制流不可达的。

检测方式：这样的代码可以很简单地利用所在方法的控制流图（CFG，即 control-flow graph）检测出来。我们只需要从方法入口开始，遍历 CFG 并标记可达语句。当遍历结束时，那些没有被标记的语句就是控制流不可达的。

#### 分支不可达代码 

在 Java 中有两种分支语句：if 语句和 switch 语句。它们可能会导致分支不可达代码的出现。

对于一个 if 语句，如果它的条件值（通过常量传播得知）是一个常数，那么无论程序怎么执行，它两个分支中的其中一个分支都不会被走到。这样的分支被称为不可达分支。该分支下的代码也因此是不可达的，被称为分支不可达代码。

检测方式：为了检测分支不可达代码，我们需要预先对被检测代码应用常量传播分析，通过它来告诉我们条件值是否为常量，然后在遍历 CFG 时，我们不进入相应的不可达分支。

### 2.2 无用赋值

一个局部变量在一条语句中被赋值，但再也没有被该语句后面的语句读取，这样的变量和语句分别被称为无用变量（dead variable，与活跃变量 live variable 相对）和无用赋值。

检测方式：为了检测无用赋值，我们需要预先对被检测代码施用活跃变量分析。对于一个赋值语句，如果它等号左侧的变量（LHS 变量）是一个无用变量（换句话说，not live），那么我们可以把它标记为一个无用赋值。

具体而言，当一个 stmt 是一个赋值语句并且它的 out fact 中找不到左值，就可以把它标记为一个无用赋值。

## 3 实验过程重点

### 判别一个语句是否是一个基本块的最后一句

在实现分支不可达代码检测时，在检测到某个分支永远不会被执行之后，需要将这个分支所在的 basic block 中的所有 stmt 都加入 dead code，在添加的时候，我们需要判断此时所遍历到的 stmt 是不是该 basic block 中的最后一条 stmt。

```java
private boolean isLastStmtInBB(Stmt stmt, CFG<Stmt> cfg) {
    if (cfg.isExit(stmt) || cfg.isEntry(stmt)) {
        return true;
    }
    // 如果该 stmt 的下一条 stmt 是 exit，那就一定是个 basic block 的结尾
    for (Stmt succ : cfg.getSuccsOf(stmt)) {
        if (cfg.isExit(succ)) {
            return true;
        }
    }
    // 如果该 stmt 的出度大于一，那就一定是个 basic block 的结尾
    if (cfg.getOutDegreeOf(stmt) > 1) {
        return true;
    }
    // 如果该 stmt 的下一条 stmt 的入度大于一，那就一定是个 basic block 的结尾
    for (Stmt succ : cfg.getSuccsOf(stmt)) {
        if (cfg.getInDegreeOf(succ) > 1) {
            return true;
        }
    }
    return false;
}
```

之中第二条规则很容易被人遗忘，因为 exit 虽然在 cfg 中被视为一个节点，但实际上并不携带任何 stmt ，我当初也没注意到，但是跑测试的时候发现了 return 语句后面怎么还有个 nop ,于是便意识到了这一点。