---
layout: post
title: ruby的Mutex与MonitorMixin
date: 2014-03-18 20:10:40
tags: ruby
categories: 技术
---

使用eventmachine和fiber的过程中发现一些以前OK的同步/锁功能的代码会出错,查资料发现ruby有两种线程间同步方式.

<!-- more -->
一种同步方式是使用Mutex,但是它有一定的局限性,见下面的代码

```ruby
require "thread"

#Mutex Example
class MutexExample

  def initialize
    super
    @mutex = Mutex.new
  end

  def first
    @mutex.synchronize do
      puts "first step"
      yield
    end
  end

  def second
    first do
      @mutex.synchronize do
        puts "second step"
      end
    end
  end

end

MutexExample.new.second
```

运行上面的代码,会出现deadlock; recursive locking (ThreadError)错误,因为Mutex是不可重入的,当线程已经获得过一次锁后,不能再次获得.

一些库项目,譬如ActiveRecord使用了Mutex来加锁,但是当我们使用fiber的时候,触发上面的场景就会报错,因为fiber是单线程的.

如何让代码在thread和fiber模式下都能良好的运行呢?

那就是使用MonitorMixin

```ruby
require "monitor"

#MonitorMixin Example
class MonitorMixinExample
  include MonitorMixin

  def initialize
    super
  end

  def first
    synchronize do
      puts "first step"
      yield
    end
  end

  def second
    first do
      synchronize do
        puts "second step"
      end
    end
  end
end

MonitorMixinExample.new.second
```
上面的代码运行不会报错,并且能够正确的输出.

redis-rb这个项目中使用的是MonitorMixin,而没有使用Mutex,在fiber模式下运行良好.

总之,如果代码可能会同时运行在thread和fiber模式下,那么还是尽量是用MonitorMixin吧!