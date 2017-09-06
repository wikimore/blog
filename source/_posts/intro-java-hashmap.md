---
layout: post
title: 说一说Java的HashMap
date: 2014-03-24 03:10:40
tags: java
categories: 技术
---

这篇文章将介绍Java7的HashMap实现.

Hash这个数据结构的实现有2种方式来解决hash冲突:`开放地址Hash法`和`链表法`.

本文不介绍`开放地址Hash法`,因为Java的HashMap是用`链表法`实现的.

<!-- more -->

下图是HashMap的属性结构

![image](/assets/img/hashmap_attr.png)

- 实现说明
  - table,一个`Entry`的`Array`,`Array`的每一位我们认为是一个`slot`,每个`slot`中可能存在零个到多个`Entry`,多个`Entry`组成一个单向链表,后加入的元素永远在链表的首位,最先加入的元素在链表尾部.
  - `Entry`存储我们的KEY和VALUE,有4个属性:key|value|next|hash,next是指向下一个`Entry`的指针,hash是key的hash值.
  - loadFactor,HashMap中Entry和slot的比值大于loadFactor时,`Array`resize,同时rehash.
  - modCount,HashMap被修改的次数,当遍历时会比较modCount来达到一旦发现修改就快速抛出异常的目的.
  - threshold,`Entry`Array的大小.
  - size,HashMap中当前元素的个数.
  - Java7的版本会根据threshold的不同,改变hash函数的hashSeed,默认是`2^31-1`

`HashMap可以理解成一个数组,而数组的每一项可能是一个单向链表.`

下图是一个loadFactor为1,slot数为6,有6个元素(Entry)的HashMap,有3个key对应的Entry哈希到了slot0上,并且组成了一个单向的链表,slot1/3/5分别哈希到了一个Entry,而slot2/4没有哈希到任何Entry.

![image](/assets/img/hashmap.png)

- PUT
  - 1.检查table是否为empty,如果为empty,则要初始化,默认初始大小是16.
  - 2.判断key是否为null,如果为null,将初始化一个hash值等于0的Entry到table的slot0中(也就是说key为null的Entry永远在slot0中).
  - 3.key不为null,计算key的hash值.
  - 4.根据hash值和table的长度计算slot的index(hash值模上table的长度).
  - 5.用当前key和其hash值与该slot的每一个Entry的key和hash比较,如果存在key和hash值都相同的,表示put一个已经存在的key,那么用当前的value替换掉Entry的oldValue,并且返回oldValue.
  - 6.如果不存在key和hash值都相同的情况,表示put了一个不存在的key,modCount加1,那么先根据loadFactor|threshold和当前HashMap的size,计算是否要对table进行resize,resize的策略是直接扩展到当前的2倍.
  - 7.之后初始化一个newEntry,newEntry的next指针指向过去slotN指向的Entry,slotN指向newEntry.

- GET
  - 1.检查table是否为empty,如果为empty,返回null.
  - 2.判断key是否为null,如果为null,到slot0中查找是否有存在key=null的Entry,存在返回Entry的value,否则返回null.
  - 3.key不为null,计算key的hash值.
  - 4.根据hash值和table的长度计算slot的index(hash值模上table的长度).
  - 5.用当前key和其hash值与该slot的每一个Entry的key和hash比较,如果存在key和hash值都相同的,返回该Entry的value,否则返回null.
  - 6.如果不存在key和hash值都相同的情况,表示该key不存在,返回null.

- REMOVE
  - 1.检查table是否为empty,如果为empty,返回null.
  - 2.计算key的hash值,然后找到对应的slot.
  - 3.遍历该slot中的每一个Entry,判断hash和key是否与要删除的相同,如果有相同的就删除,并返回该Entry,否则返回null
  - 4.判断返回的Entry是否是null,不是就返回Entry的value.

其他的API方法也大同小异,在这里不做详细的说明.

- 使用注意
  - 不可并发put,会造成死循环,如果要并发put,可以使用`ConcurrentHashMap`.
  - 遍历时不可修改,会抛出`ConcurrentModificationException`.
  - 如果遍历value,使用`entrySet`,不要使用`keySet`,减少遍历次数.
  - `putAll`要注意,有时会造成2次resize.