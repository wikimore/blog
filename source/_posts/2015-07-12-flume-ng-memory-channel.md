---
layout: post
title: Flume-ng中的MemoryChannel
date: 2015-07-12 00:00:00
tags: flume
categories: 技术
---

### 00 概述
MemoryChannel是使用纯内存缓冲Event的Channel实现，所以速度上比较快速，容量不会太大，可靠性不够，所以适用一些可以丢数据，但对性能要求较高的业务。

### 01 实现原理
主要类：`MemoryChannel`和`MemoryTransaction`

MemoryChannel使用LinkedBlockingDeque类型的queue来存储Event，但是单一个queue没法实现Transaction；

MemoryTransaction实现Transaction语义，MemoryChannel与MemoryTransaction是一对一到多的关系，MemoryTransaction使用ThreadLocal，每个Thread有自己MemoryTransaction。

MemoryTransaction分别有一个putList和takeList，都是LinkedBlockingDeque，putList用来暂存新增的未提交的Event，如果commit，putList的Event会被顺序的放入queue中，rollback的话就清空putList，takeList用来暂存刚被取走但未提交的Event，如果commit，就清空takeList，rollback的话就会把takeList的Event再移回到queue里。

### 02 问题
MemoryChannel这样的实现很简单，逻辑上也没啥问题，但是在看源码的过程中，发现几个可以优化的地方

- MemoryTransaction做take/commit/rollback操作时，都会synchronized同一个对象
- 移动Event使用循环的poll和offer，没有使用drainTo

LinkedBlockingDeque是Thread-Safe，MemoryTransaction也不存在并发问题(ThreadLocal)，对同一个Thread来说，不可能同时take/commit/rollback，而synchronized即使没有锁貌似也会有一些额外的操作，性能会差一点点

官方描述drainTo有一句话`This operation may be more efficient than repeatedly polling this queue`

### 03 总结
MemoryChannel很简单，但是官方的实现并不完美，应该有优化的空间，感觉支持Transaction也有点鸡肋。

