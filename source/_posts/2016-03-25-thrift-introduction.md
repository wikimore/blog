---
layout: post
title: Thrift简介
date: 2016-03-25 15:10:40
tags: [rpc,thrift]
categories: 技术
---

Thrift是ASF支持下的一个跨语言的RPC框架，类似的有avro、protobuf等，最大的优点应该就是支持的语言特别多。

使用Thrift最关键的一点应该是要弄懂他的IDL(Interface Description Language)，[官方的文档](http://thrift.apache.org/docs/idl)还是有比较详细的说明的，主流的数据结构都有支持，也支持结构体、共同体、枚举类型，但是不提供继承关系(序列化的原因)，唯一可以继承的就是service，这样可以写一些通用的东西到一个service中，然后其他的业务service可以继承该service，还有就是可以自定义exception。

Thrift项目主要分为2个部分

- Compiler:负责IDL解析以及不同语言代码的生成，由C++编写，基于yacc和bison做语法分析，抽象出Generator生成不同语言的原生代码
- Library:各个不同语言的运行时类库

下面主要介绍Java版本的Library。

Thrift的Library大致上可以分成几个部分：

- Transport:网络连接抽象，包括InputStream和OutputStram
- Protocol:序列化协议，读写数据到Transport中，包括原生类型和结构化类型的对象
- Processor:服务端接收请求的处理器，负责选择调用方法对应的Function
- Function:由Processor管理，对应一个IDL中定义的方法，内部调用对应方法的实现，框架定义整体的处理流程(模板模式)，业务逻辑需要开发人员实现
- Client:客户端，依赖Protocol，框架只定义抽象，实现部分是根据IDL自动生成的(参数序列化和结果反序列化)

下图展示了各个抽象之间的依赖关系和所处的层次。

![image](/assets/img/thrift01.png)

通过上图可以了解整个Thrift工作的大体流程，至于如何安装使用，官方有很详细的[说明](http://thrift.apache.org)，个人觉得Thrift是一个非常不错的RPC框架。

