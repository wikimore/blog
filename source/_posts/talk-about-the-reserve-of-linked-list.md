---
title: 聊聊链表的翻转
tags:
  - 链表
  - 数据结构
  - 面试
categories:
  - 技术
date: 2022-10-25 11:14:29
---

### 00 概述
最近在准备技术面试，在这里就记录和总结一下我对链表翻转的一些理解。

为什么会想到记录这个呢？

虽然链表的翻转不是什么复杂的问题，但是在看一些笔试题的时候我发现有一些复杂问题拆解下来就会涉及到链表的翻转，是一个很基础的知识点，有必要掌握牢靠。

什么是链表的翻转？举个例子

```
初始链表list A：2->1->3->4->5 翻转后 list A：5->4->3->1->2
```

通俗点说就是把链表元素的顺序倒过来排序。要达成什么效果相信大家已经清楚，下面就来说说如何实现这样的效果，总结下来大概有以下几种方法。

### 01 迭代法
正常大家最顺滑想到的办法就是从左到右依次遍历这个链表，然后把指针的指向翻过来，比如 `list A：a->b->c->d->e`变成 `list A：a<-b<-c<-d<-e`，如下图

<img src="/assets/img/linkedlist00.svg" width = "100%" alt="linkedlist00" align="center" />


```Java
// 定义3个指针
prev = null;
cur = head;
next = head.next;

// 从头开始遍历
while(cur != null){
	cur.next = prev;
	prev = cur;
	cur = next;
	next = next.next;
}


```

### 02 递归法
递归是计算机经常需要考虑的方法，与上面边迭代边调整指针的方式不同，递归是在退出迭代的过程中做指针变化的，理解起来稍微有点麻烦，但是代码可能更加简单一些。先看一段伪代码

```Java
public Node reserve(Node head){
	if(head != null && head.next == null){
		return head;
	}
	// 递归
	Node newHead = reserve(head.next);

	// 指针变化  a->b->null ----->   null<-a<-b
	head.next.next = head; 
	head.next = null;
	return newHead;
}
```
我们必须理解到，递归流程下，在栈退出前，才做指针的变化。就向下图一样，有一个先进栈的过程。
<img src="/assets/img/linkedlist01.svg" width = "100%" alt="linkedlist00" align="center" />

### 03 头插法
依然使用迭代的方式，类似于迭代法，只是指针的变化方式不一样，头插法的思想是构建一个新的链表头，遍历当前链表，插入到新的链表的头部。伪代码如下：

```Java
// 定义3个指针
newHead = null;

// 从头开始遍历
while(head != null){
	cur = head;
	head = head.next;
	cur.next = newHead;
	newHead = cur;
}
return newHead;
```

### 04 总结
其他还有`原地逆置法`，我理解其实也是迭代法的一个子集，我们并不需要知道`茴`的所有写法，关键是基于自己的认知，掌握一个自己最容易理解的指针变化逻辑。