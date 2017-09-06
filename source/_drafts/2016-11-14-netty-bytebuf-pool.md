---
layout: post
title:  "Netty的ByteBuf对象池"
date:   2016-11-14 23:32:40
tags: [netty]
categories: 技术
---
## 介绍
对象池是Netty提供的一个重要的功能，能够有效的减少内存分配的时间，减少总体的GC时长。

## 原理

PooledByteBufAllocator是池对象的分配器，内部包含两个`PoolArena`的数组，分别用来分配和管理堆内和堆外内存。

### 参数
PooledByteBufAllocator有一些重要的配置参数：

#### DEFAULT_PAGE_SIZE

默认的内存页大小，可以通过`-Dio.netty.allocator.pageSize`来设置，最小必须>=4096，而且必须是2的指数倍，默认值为8192，即8KB。

#### DEFAULT_MAX_ORDER

默认的最大偏移值，可以通过`-Dio.netty.allocator.maxOrder`来设置，范围在[0~14]之间，默认值是11。

#### DEFAULT_CHUNK_SIZE

默认每个`PoolChunk`的大小，`chunkSize = DEFAULT_PAGE_SIZE << DEFAULT_MAX_ORDER`，默认值为16777216，即16MB。

#### DEFAULT_NUM_HEAP_ARENA/DEFAULT_NUM_DIRECT_ARENA

默认`PoolArena`的个数。可以通过`-Dio.netty.allocator.numHeapArenas`和`-Dio.netty.allocator.numDirectArenas`来设置。
默认值为`Math.min(Runtime.getRuntime().availableProcessors() * 2,Runtime.getRuntime().maxMemory() / DEFAULT_CHUNK_SIZE / 2 / 3)`。4G内存大概可以支持42个`PoolArena`。

#### DEFAULT_TINY_CACHE_SIZE

默认`Tiny`级别内存区域的个数，可以通过`-Dio.netty.allocator.tinyCacheSize`来设置，默认是512个。

#### DEFAULT_SMALL_CACHE_SIZE

默认`Small`级别内存区域的个数，可以通过`-Dio.netty.allocator.smallCacheSize`来设置，默认是256个。

#### DEFAULT_NORMAL_CACHE_SIZE

默认`Normal`级别内存区域的个数，可以通过`-Dio.netty.allocator.normalCacheSize`来设置，默认是64个。

#### DEFAULT_MAX_CACHED_BUFFER_CAPACITY

#### DEFAULT_CACHE_TRIM_INTERVAL

### 内存分配大小原则

`PoolArena`具体会根据申请的内存大小，分配对应大小区域的内存空间，不同的大小有不同的规则。

假设申请内存的大小为reqCapacity

- reqCapacity >= chunksize，直接分配reqCapacity的内存
- chunksize > reqCapacity > 512Bytes，分配最小的大于reqCapacity的2的指数倍的内存，如reqCapacity=513Bytes，分配1024Bytes,如reqCapacity=1023Bytes，分配1024Bytes,如reqCapacity=12768Bytes，分配16384Bytes
- reqCapacity <= 512Bytes，并且是16Bytes的整数倍，分配reqCapacity的内存
- reqCapacity <= 512Bytes，且reqCapacity不是16Bytes的整数倍，分配比reqCapacity大的最小的16Bytes的整数倍的内存，如reqCapacity=33Bytes，分配48Bytes,reqCapacity=47Bytes，分配48Bytes,reqCapacity=488Bytes，分配496Bytes

具体为什么按照这个规则？是因为

### 内存分配区域有哪些

我们知道`PoolArena`是内存池最大的一个单元，在`PoolArena`内部会有多个区域类型，在这些区域中，实际拥有内存的是`PoolChunk`，虚拟的包括`PoolChunkList`、`TinyPoolSubpage`、`SmallPoolSubpage`

### 内存分配区域原则

先获取PoolThreadCache，这是一个ThreadLocal变量，如果没有会初始化，PoolThreadCache内部会绑定一个PoolArena，这个线程以后所有的内存分配都通过这个PoolThreadCache以及其内部绑定的PoolArena。

之后会开始正式分配内存，首先根据申请的内存大小算出一个符合分配规则的实际分配内存大小，一般会是2的指数倍，因为这样方便计算、申请和回收。

算出实际要分配的内存大小之后开始正式分配内存，如果小于pagesize，分为两种情况，一种是分配tiny大小的内存，另一种是分配small大小的内存。区别是分配的大小，逻辑流程是类似的，下面以tiny为例。

首先PoolThreadCache内部有一个MemoryRegionCache这样的缓存，分为tiny、small、normal三种，并且每种又分direct和heap，那么就是总共6个MemoryRegionCache。

这个MemoryRegionCache是做什么的呢？其内部有一个队列，队列里是一个PoolChunk和handle对象，PoolChunk很好理解，就是一个大的内存区域，handle是分配的内存在整个PoolChunk的offset和length，高32位是xxx，低32位是xxx。





如果申请内存>CHUNK_SIZE，直接分配
如果申请

Netty的对象池参考了`jemalloc`等主流的内存分配库，最大的区域叫做`PoolArena`，Netty默认初始化长度为1的`PoolArena`数组。

`PoolArena`内部包含多个`PoolChunk`