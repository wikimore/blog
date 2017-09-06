---
layout: post
title: 编写自己的maven插件
date: 2014-12-27 16:38:40
tags: maven
categories: 技术
---
Maven是一个Java语言编写的项目依赖和构建工具。
<!-- more -->

Maven常用命令包括：

- clean
- test
- compile
- package
- install
- deploy

> 当然还有一些不是特别常用的命令，如：verify，site等

上面这些命令的执行就依赖于各种不同`maven-plugin`。

每个命令都会包含一个或者几个`Lifecycle`，每个`Lifecycle`有可能会默认的绑定到某一个(或几个)插件的某一个`goal`上。

这里有两个概念`Lifecycle`和`goal`：

- `Lifecycle`默认由Maven定义，默认`Lifecycle`详见[Lifecycle_Reference](http://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html#Lifecycle_Reference)；
- 而`goal`代表一个目标(完成一个任务)，每个以插件可以有一个或者多个`goal`。


> 譬如clean就包含3个Lifecycle
> 
>  - pre-clean
>  - clean
>  - post-clean
>  
>  而`Lifecycle`clean则内建绑定到`maven-clean-plugin`的clean这个`goal`上
>
> 更多详情请见[Built-in_Lifecycle_Bindings](http://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html#Built-in_Lifecycle_Bindings)
> 

那么如何编写一个Maven的插件呢？

- 建立一个Maven的项目，一般项目名称就会是插件的名称，
> 项目的命名是有一定的讲究的，maven内建的插件名称都是`maven-xxx-plugin`这样的模式，而我们的插件名称一般建议叫做`xxx-maven-plugin`，这个在maven的官网上有说明。
- 修改pom的`packaging`类型为maven-plugin
> 打包流程上会有不同
- 增加依赖
> 根据插件功能的不同要依赖不同的jar
- 增加插件依赖
> maven-plugin-plugin
- 编写自己的类XxxxMojo，并继承`AbstractMojo`
- 在类上增加Annotation`@Mojo`来说明XxxxMojo：
	- name：`goal`的名称
	- defaultPhase：默认绑定的`Lifecycle`
- 实现execute方法
- 测试(可以使用JUnit，也有插件可以使用)
- 打包发布

这样一个简单的maven插件就开发完成([我的Maven插件Sample工程](https://github.com/wikimore/sample-maven-plugin))，但是要开发一个可用的插件，还有很多工作要做。

> 参考：
> 
> - [sonatype的maven说明](http://books.sonatype.com/mvnref-book/reference/)
> - [Github上的Apache的maven插件源码](https://github.com/apache/maven-plugins)

