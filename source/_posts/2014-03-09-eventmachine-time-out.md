---
layout: post
title: 关于eventmachine的timeout
date: 2014-03-10 02:10:40
tags: ruby
categories: 技术
---

工作中负责实现thrift的eventmachine客户端版本,在实现过程中就遇到了读写timeout如何实现的问题.

eventmachine的connection有两个连接的参数`pending_connect_timeout`和`comm_inactivity_timeout`,但就是没有read和write的timeout的配置参数,这样的话,业务调用就没有超时,根本不能够在线上使用.

后来发现em有`Timer`功能,那么就在每次调用read的时候,同时初始化一个`Timer`,利用`Timer`来控制超时.

```ruby
def read(size,timeout)
  timer = nil
  if timeout and timeout > 0
    timer = EventMachine.add_timer(timeout){
      // timeout_callback
    }
  end
  .
  .
  // read code
  .
  .
  if timer
    EventMachine.cancel_timer(timer)
  end
end
```

经过测试,这个方案还是可以的,我本来以为这样会增加CPU的消耗,但是测试并没有发现CPU占用有增长(应该是em的轮询线程就一个mainloop的原因,Timer多了只是增加了轮询的个数),就是每次都会初始化一个`Timer`对象,感觉有点坑爹.但是也没有什么好办法,而且后来发现[Thrift官方的EM支持](https://issues.apache.org/jira/browse/THRIFT-146)就是这个方案来设置超时的,所以就在线上使用了.

后来和老大讨论了这个问题,老大也觉得挺坑爹,给我一个建议,能否使用`Period_Timer`,他发现Java的so_timeout不是设置SO_RCVTIMEO实现的,见[SocketInputStream.c](http://hg.openjdk.java.net/jdk7u/jdk7u60/jdk/file/c4763416f516/src/solaris/native/java/net/SocketInputStream.c),其中使用了NET_timeout函数,感觉上像是循环调用select/poll的api,如果在timeout时间范围内fd还不是readable,那么就超时,否则读数据(不过这样我就不明白如何确定数据读完).

通过这个问题,感觉自己有时间得去看看epoll/kqueue/select/poll等unix网络相关的内容.以后会记录这个问题的详细研究结果.