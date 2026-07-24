+++
date = '2026-07-26T19:03:42+08:00'
draft = false
categories = ["Static Analysis"]
tags = ["tai-e", "assignment"]
title = 'Tai-e Assignment 6: 上下文敏感的指针分析'
+++

## 1 实验内容

- 为 Java 实现一个上下文敏感的指针分析框架。
- 作为指针分析的一部分，随着指针分析一起实现调用图（call graph）构建。
- 实现几种常见的上下文敏感策略（context sensitivity variants）。

## 2 实验概览

## 3 实验过程重点

没啥重点，其实理解起来不算困难，但是总是会在各种上下文之间迷失方向，尤其是当有很多个上下文要处理的时候，经常会把上下文填错。

### 关于对象、类型敏感

要注意在实现的时候是将 recv 相关的对象/类型加入 recv 所携带的上下文，而不是 callSite 所带的上下文。

```java
public Context selectContext(CSCallSite callSite, CSObj recv, JMethod callee) {
    return addContext(recv.getContext(), recv.getObject().getContainerType());
}
public Context selectContext(CSCallSite callSite, CSObj recv, JMethod callee) {
    return addContext(recv.getContext(), recv.getObject());
}
```