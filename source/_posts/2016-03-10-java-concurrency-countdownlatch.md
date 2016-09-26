---
layout: post
title: Java并发之CountDownLatch
date: 2016-03-10 00:00:00
tags: [java,concurrency]
categories: 技术
---
CountDownLatch是juc包下一个非常简单的并发辅助类，提供线程间等待的支持，如一组线程A等待另外一组线程B完成某些操作后(一组可以是一个或者多个线程)，才能继续执行时，可以使用CountDownLatch来完成控制。
<!-- more -->
CountDownLatch主要提供两个方法：`await`和`countdown`，内部类`Sync`继承了AQS，实现类似共享模式的同步机制。

`await`从字面上理解就是等待，当然实现也确实是等待，实现通过调用内部的sync对象来等待。

假如线程组B中有一个出现问题，没有调用`countdown`方法，那个线程组A中的线程就没法继续执行，所以`await`还有一个重载方法`await(long,timeunit)`，也就是当等待给定时长后，即使B中的线程有些没有执行完毕，但是A线程组中的线程再不等待，继续执行后面的逻辑。

CountDownLatch初始化需要提供一个count，`countdown`每次被调用，count都会减1，当count等于0时，所有`await`的线程会依次被唤醒。

下面的例子有三个角色，Leader、Developer、Tester，Leader管理一组Developer和Tester，Leader设计好系统，Developer开始编写代码，Tester同时开始编写测试用例，Tester等待Developer开发好后，执行测试用例。

```
public class Leader {
  public static void main(String[] args) throws Throwable {
    CountDownLatch startSignal = new CountDownLatch(1);
    CountDownLatch developerDoneSignal = new CountDownLatch(8);
    CountDownLatch testerDoneSignal = new CountDownLatch(8);

    // 找到测试人员
    // 找到开发者
    for (int i = 0; i < 8; ++i) {
      new Thread(new Tester(startSignal, developerDoneSignal, testerDoneSignal)).start();
      new Thread(new Developer(startSignal, developerDoneSignal)).start();
    }

    // 设计系统
    System.out.println("design");

    // 设计好后，开发和测试开始工作
    startSignal.countDown();
    startSignal.countDown();

    // 等待测试完成
    testerDoneSignal.await();

    // 完成
    System.out.println("done");
  }

  static class Developer implements Runnable {
    private final CountDownLatch startSignal;
    private final CountDownLatch doneSignal;

    Developer(CountDownLatch startSignal, CountDownLatch doneSignal) {
      this.startSignal = startSignal;
      this.doneSignal = doneSignal;
    }

    public void run() {
      try {
        // 等待Leader设计
        startSignal.await();
        // 开发
        System.out.println("Develop");
        Thread.sleep(5);
      } catch (InterruptedException ex) {
        // ignore
      } finally {
        // 完成后通知测试人员
        doneSignal.countDown();
      }
    }
  }

  static class Tester implements Runnable {
    private final CountDownLatch testCaseStartSignal;
    private final CountDownLatch testStartSignal;
    private final CountDownLatch doneSignal;

    Tester(CountDownLatch testCaseStartSignal, CountDownLatch testStartSignal, CountDownLatch doneSignal) {
      this.testCaseStartSignal = testCaseStartSignal;
      this.testStartSignal = testStartSignal;
      this.doneSignal = doneSignal;
    }

    public void run() {
      try {
        // 等待Leader设计
        testCaseStartSignal.await();
        // 编写测试用例
        System.out.println("Write Test case");
        Thread.sleep(5);
        // 等待开发完成
        testStartSignal.await();
        // 测试
        System.out.println("Test");
        Thread.sleep(5);
      } catch (InterruptedException ex) {
        // ignore
      } finally {
        // 测试完成通知Leader
        doneSignal.countDown();
      }
    }
  }
}
```

```
design
Write Test case
Develop
Write Test case
Develop
Write Test case
Develop
Write Test case
Write Test case
Develop
Develop
Develop
Write Test case
Write Test case
Develop
Write Test case
Develop
Test
Test
Test
Test
Test
Test
Test
Test
done
```
从输入可以看出，Leader设计完成后，Developer和Tester才能开始工作，编码和编写测试用例是并行执行的，但是必须开发完毕后，测试才能进行测试。

总结，通过CountDownLatch可以组织不同的线程的执行顺序，确保前置任务完成，后续任务才能开始。