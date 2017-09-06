---
layout: post
title: 排序算法之HeapSort
date: 2016-03-03 00:00:00
tags: 排序算法
categories: 技术
---
堆排序是一种基于比较的排序算法，可以简单的认为是一种改进的选择排序，虽然在实际中比快速排序要慢，但是其优势是最坏情况基本也是O(n log n)的时间复杂度，空间复杂度是O(1)，也就是不需要额外的内存空间。堆排序是稳定的排序算法。
<!-- more -->
理解堆排序首先要了解什么是堆，堆分为大根堆和小根堆，如果根节点是最小的数，为小根堆，根节点是最大的数，为大根堆，堆一般使用一维的数组来表示，可以看做是一个完全的二叉树：

- 树的根节点是array[0]
- 每个节点的左子节点为array[2i + 1]
- 每个节点的右子节点为array[2i + 2]
- 每个节点的父节点为array[floor((i - 1) / 2)]

堆排序算法可以被分成两个部分：

- 建立大(小)根堆
- 堆排序

以上两部操作都涉及到`堆调整`，调整的目的是建立完全最大(小)根堆。

建堆就是建立一个大根堆或者小根堆，以大根堆为例，假设有下列数组

```
[1,4,7,6,11,5,8,2,10,9,3]
```
初始堆如下图：

![image](/assets/img/heapsort001.png)

首先我们要建立`完全最大堆`，起始选取最后一个叶子节点的父节点，假设该节点为`array[i]`，本例`i = 4`，比较该节点与其子节点，如果子节点比其大，选取最大的子节点与其进行交换，该例子中`11 > 9 > 3`，所以不需要交换，然后比较`array[i--]`，同样的逻辑，这时`6 > 2 < 10`，所以交换6和10，变换后如下图(蓝色表示交换过位置)

![image](/assets/img/heapsort002.png)

之后继续比较`array[i--]`，那么7和8要交换位置，变化结果如下

![image](/assets/img/heapsort003.png)

之后继续比较`array[i--]`，4和11要交换位置，变化结果如下

![image](/assets/img/heapsort004.png)

因为4和11交换后，4作为父节点，不一定满足完全大根堆，所以要比较4和其子节点，如果子节点大的话，需要交换，那么这一步的变换如下
![image](/assets/img/heapsort005.png)

之后继续比较`array[i--]`，1和11要交换位置，变化结果如下
![image](/assets/img/heapsort006.png)

因为1和11交换后，1作为父节点，不一定满足完全大根堆，所以要比较1和其子节点，如果子节点大的话，需要交换，那么这一步的变换如下
![image](/assets/img/heapsort007.png)
之后同理，1沉到最下面
![image](/assets/img/heapsort008.png)

到此为止，建堆的过程就结束了。

接下来要做`堆排序`，过程简单描述就是堆顶元素和数组最后一个元素交换位置，交换后，对二叉树进行剪枝，排除最后一个元素，然后从堆顶开始调整堆，重复上面的过程。

变换图如下(绿色表示剪枝)
![image](/assets/img/heapsort009.png)
![image](/assets/img/heapsort010.png)
![image](/assets/img/heapsort011.png)
![image](/assets/img/heapsort012.png)
![image](/assets/img/heapsort013.png)
![image](/assets/img/heapsort014.png)
![image](/assets/img/heapsort015.png)
![image](/assets/img/heapsort016.png)

到目前位置最后两位的数据排序是正确的，后面继续该流程就完成了整个堆排序的过程，这里就不继续贴图了。

> 参考
> [堆排序-Wikipedia](https://zh.wikipedia.org/wiki/%E5%A0%86%E6%8E%92%E5%BA%8F)
> [heapsort-Wikipedia](https://en.wikipedia.org/wiki/Heapsort)
> [Sorting_algorithm#Stability](https://en.wikipedia.org/wiki/Sorting_algorithm#Stability)

