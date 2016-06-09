---
title: "Java并发之Lock与synchronized"
tags: [java,concurrency]
categories: 技术
---

在Java中，线程同步可以用2中方式解决：`synchronized`和`Lock`，那么这2种方式在内部原理上有什么不同呢？本文将详细说明。

对于操作系统而言，实现同步或者说锁的方式只有pv信号量