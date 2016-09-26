---
layout: post
title: Java并发之Semaphore
date: 2016-03-12 00:00:00
tags: [java,concurrency]
categories: 技术
---
Semaphore是juc包下一个并发辅助类，官方定义为`可以计数的信号量`，允许N个获得`许可`的线程同时访问某个资源或执行某段代码，`许可`可以被归还，归还后的`许可`可以被其他线程获取，如果`许可`不足，线程就等待，当其他线程归还`许可`，等待的线程被唤醒，尝试获取`许可`。
<!-- more -->
> 注意：许可可以被同一个线程获取多次，也能够被同一个线程归还多次。

当资源有限时，并发获取使用Semaphore控制是一个非常好的方式，比如数据库连接池，网络连接池等。

Semaphore主要方法有：

- `acquire`：阻塞获取`许可`，直到获得许可为止，响应`interrupt`
- `acquireUninterruptibly`：阻塞获取`许可`，直到获得许可为止，不响应`interrupt`
- `tryAcquire`：尝试获取`许可`，返回`boolean`类型，true表示获得，false未获得
- `release`：释放(归还)`许可`，同时唤醒所有尝试获取`许可`的线程

当然还有一些其他的方法，不过语义上大同小异。

Semaphore内部通过继承AQS，预先申请`许可`，同时实现类`Fair`和`Nonfair`的等待唤醒机制

> 同CountDownLatch类似，都是通过AQS来实现内部的流程，juc下面还有不少工具也是通过AQS来实现，之后会详细介绍。

```
public class Pool {
  private final Stack<Connection> stack;
  private final Semaphore semaphore;

  public static void main(String[] args) {
    Pool pool = new Pool(2);
    Thread[] threads = new Thread[8];
    for (int i = 0; i < 8; i++) {
      Thread t = new Thread(new Runnable() {

        @Override
        public void run() {
          Connection connection = pool.borrowConnection();
          System.out.println(Thread.currentThread().getName() + " borrowed " + connection);
          pool.returnConnection(connection);
          System.out.println(Thread.currentThread().getName() + " return " + connection);
        }
      });
      threads[i] = t;
    }
    for (int i = 0; i < 8; i++) {
      threads[i].start();
    }
  }

  public Pool(int size) {
    // 初始化许可
    stack = new Stack<Connection>();
    for (int i = 0; i < size; i++) {
      stack.push(new Connection(i));
    }
    semaphore = new Semaphore(size);
  }

  public Connection borrowConnection() {
    // 尝试获取许可，如果没有就阻塞
    semaphore.acquireUninterruptibly();
    // 获取连接
    return stack.pop();
  }

  public void returnConnection(Connection connection) {
    // 返回连接
    stack.push(connection);
    // 归还许可，其他线程可以获得该许可
    semaphore.release();
  }

  static class Connection {
    private final int id;

    public Connection(int id) {
      this.id = id;
    }

    @Override
    public String toString() {
      StringBuilder builder = new StringBuilder();
      builder.append("Connection [id=").append(id).append("]");
      return builder.toString();
    }

  }

}
```

下面是输出，可以看到获取连接的线程是没有顺序的

```
Thread-0 borrowed Connection [id=0]
Thread-6 borrowed Connection [id=0]
Thread-6 return Connection [id=0]
Thread-0 return Connection [id=0]
Thread-7 borrowed Connection [id=0]
Thread-7 return Connection [id=0]
Thread-1 borrowed Connection [id=0]
Thread-1 return Connection [id=0]
Thread-2 borrowed Connection [id=0]
Thread-2 return Connection [id=0]
Thread-3 borrowed Connection [id=0]
Thread-3 return Connection [id=0]
Thread-4 borrowed Connection [id=0]
Thread-4 return Connection [id=0]
Thread-5 borrowed Connection [id=0]
Thread-5 return Connection [id=0]
```

如果我们把`Pool`的构造方法中的

```
semaphore = new Semaphore(size);
```

改成

```
semaphore = new Semaphore(size, true);
```

输出如下

```
Thread-0 borrowed Connection [id=0]
Thread-0 return Connection [id=0]
Thread-1 borrowed Connection [id=0]
Thread-1 return Connection [id=0]
Thread-2 borrowed Connection [id=0]
Thread-2 return Connection [id=0]
Thread-3 borrowed Connection [id=0]
Thread-3 return Connection [id=0]
Thread-4 borrowed Connection [id=0]
Thread-4 return Connection [id=0]
Thread-5 borrowed Connection [id=0]
Thread-5 return Connection [id=0]
Thread-6 borrowed Connection [id=0]
Thread-6 return Connection [id=0]
Thread-7 borrowed Connection [id=0]
Thread-7 return Connection [id=0]
```
获取连接的顺序固定了，这就是公平和非公平的区别，公平策略会先到先得，内部是一个队列，而非公平的策略并非先到先得，内部是一个自旋的过程，通俗说就是有人插队^_^。

Semaphore可以提供`许可`，让有限的资源被多个线程共享使用。