---
layout: post
title: Java并发之CyclicBarrier
date: 2016-03-11 00:00:00
tags: [java,concurrency]
categories: 技术
---
CyclicBarrier是juc包下一个并发辅助类，类似于CountDownLatch，但又不同，CyclicBarrier保证一组线程在同一个地方互相等待，直到最后一个线程到来后，然后一起再继续向下执行。
<!-- more -->
CyclicBarrier提供了一个叫做`Generation(代/批)`的概念，每个`Generation`的线程由该`Generation`最后一个到达的唤醒其他先到的线程，一个CyclicBarrier允许有N个`Generation`。

CyclicBarrier初始化支持两个参数:

- parties，int类型，标记多少个线程等待后，全部唤醒继续执行
- barrierAction，Runnable类型，如果不为null，最后一个到达barrier的线程会执行此Runnable

最主要方法`await`，内部主要依赖`Lock`、`Condition`以及一个`count`计数来工作，`count`初始值为构造参数parties的值

- `Lock`保证线程同步，同一时刻只有一个线程能够执行await的核心逻辑
- 核心逻辑先做`--count`操作，然后判断`count计数`的值是否为0
  - 如果不是，在`Condition`下等待
  - 如果是，执行barrierAction，然后唤醒所有等待在`Condition`的线程，最后新初始化一个`Generation`，并将parties的值赋值给`count`(重置计数)

> 注意：如果parties=2，不要同时启动4个线程，因为并不能完全保证不同`Generation`的线程是顺序唤醒的，因为内部公用同一把`Lock`

代码示例：模拟轮渡载满才过河

```
public class Ferry {

  public static void main(String[] args) throws Throwable {
    int count = 4;
    CyclicBarrier cyclicBarrier = new CyclicBarrier(count, new Runnable() {

      @Override
      public void run() {
        System.out.println("船开了");
      }
    });
    Thread[] threads = new Thread[count];
    for (int i = 0; i < count; i++) {
      Thread t = new Thread(new Runnable() {

        @Override
        public void run() {
          try {
            System.out.println(Thread.currentThread().getName() + "等待过河");
            cyclicBarrier.await();
            System.out.println(Thread.currentThread().getName() + "过河成功");
          } catch (InterruptedException e) {
          } catch (BrokenBarrierException e) {
          }
        }
      });
      threads[i] = t;
    }
    for (int i = 0; i < count; i++) {
      threads[i].start();
    }

    Thread.sleep(10000);
  }
}
```

输出如下：

```
Thread-3等待过河
Thread-2等待过河
Thread-0等待过河
Thread-1等待过河
船开了
Thread-1过河成功
Thread-0过河成功
Thread-2过河成功
Thread-3过河成功
```
可以发现最后到达的线程会先执行(这个毋庸置疑)，其他线程随机顺序，这主要是因为内部的`Lock`使用的是非公平的，所以并非先到先执行。

CyclicBarrier可以保证多个线程在某一处互相等待，然后继续执行，在某些特定的场景下使用会非常方便、有效。