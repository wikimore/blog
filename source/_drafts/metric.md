---
layout: post
title:  "如何监控"
date:   2014-04-27 15:10:40
tags: [linux]
categories: 技术
---

今天看yammer的Metrics,发现里面有一个Meter统计就是参考*nix的LA来做的,正好以前面试也遇到过类似的问题,就查看了一下,发现使用类似[指数加权移动平均值](http://en.wikipedia.org/wiki/Moving_average#Exponential_moving_average)来计算该值。

*nix系统可以通过uptime命令来查看load average,有3个数值,从做到右分别表示最近1分钟,5分钟,15分钟服务器的负载状况,越大代表负载越高。

这个数字是表示最近一个周期内(1/5/15)CPU的等待队列和执行队列中任务的个数。

所以,当系统的LA过高,可能真的是CPU不够(正在处理的任务过多),也可能是IO不够(等待处理的任务过多)造成的。


