---
layout: post
title: Thrift协议介绍
date: 2016-04-04 15:10:40
mathjax: true
mathjax2: true
tags: [rpc,thrift]
categories: 技术
---
本文主要说明Thrift如何序列化和反序列化数据。主要分析为什么在客户端调用method，服务端会执行对应的method，同时客户端能够获取到返回值，中间都发生了什么。

Thrift序列化的核心类是`TProtocol`，这是一个抽象类，主要实现有：
- `TBinaryProtocol`：二进制协议
- `TCompactProtocl`：带压缩的二进制(其实只对i16、i32、i64这3种类型以及field的编号进行压缩，详细可以搜索zigzag编码)

本文以`TBinaryProtocol`为例说明，协议如下所示

```
----------------------------------------------------------------------------------------
|                              header                            |        body         |
|  magic  | method name length |  method name  | sequence number |       result        |
|    4    |         4          | N length size |        4        |          X          |   
----------------------------------------------------------------------------------------
```
总体我们可以分为`header`和`body`。

`header`首先是4个字节的magic，Thrift协议的magic是一个32位的数字，高16位是`8001`(个人理解代表版本号，目前是第一个版本)，低16位根据TMessageType得到，目前TMessageType分为4种：

- CALL = 1 调用消息，如`0x80010001`
- REPLY = 2 应答消息，如`0x80010002`
- EXCEPTION = 3 异常消息，如`0x80010003`
- ONEWAY = 4 单向消息，属于调用消息，但是不需要应答，如`0x80010004`

之后4个字节表示方法名称的长度N，之后是N方法名称的字节，最后是4个字节的序列号。

`body`首先是一个`struct`类型，内部会根据类型的不同各不相同，同时支持不同类型的嵌套，目前支持的类型包括：

- byte
- bool
- short
- int
- long
- double
- string
- bytearray
- map
- list
- set
- field
- struct

> 注意：`exception`可以理解为struct的一种，按照struct来序列化和反序列化。

`byte`占1个字节，`bool`也是占用1个字节，true=1，false=0。

`short`占2个字节，`int`占4个字节，`long`和`double`都是占用8个字节。

`string`和`bytearray`类似，占4+N字节，描述如下：

```
--------------------
| size |  content  |
|   4  |     N     |
--------------------
```
string使用UTF-8编码成byte数组。

`map`占`1+1+4+N*X+N*Y`个字节，描述如下：

```
----------------------------------------------------------------------------------------
|  key-type  | value-type |  size  | key1 | value1 | key2 | value2 |...| keyN | valueN |
|      1     |      1     |    4   |   X  |   Y    |   X  |   Y    |...|   X  |   Y    |
----------------------------------------------------------------------------------------
```
其中key-type和value-type可以是任何以上的类型。

`list`和`set`类似，占`1+4+N*X`个字节，描述如下：

```
----------------------------------------------------------------
|  element-type  |  size  | element1 | element2 |...| elementN |
|        1       |    4   |    X     |    X     |...|    X     |
----------------------------------------------------------------
```
其中element-type可以是任何以上的类型。

`field`占`1+2+X`个字节，描述如下：

```
---------------------------------------
| field-type | field-id | field-value |
|      1     |     2    |      X      |
---------------------------------------
```
`field`不是独立出现，而是出现在`struct`内部，其中field-type可以是任何以上的类型，field-id就是定义IDL时该`field`在`struct`的编号，field-value是对应类型的序列化结果。

`struct`占`X`个字节，这个不太好估计，需要具体定义具体计算，可以按照入下描述：

```
--------------------------------
| field1 | field2 |...| fieldN |
|    M   |    M   |...|    M   |
--------------------------------
```

在使用上有些需要注意的点：

- 对于方法定义了返回值为`struct`、`map`、`set`、`list`，如果服务端返回null，thrift是不能够很好的支持的(会报错)，大家可以根据具体情况返回empty的对象或者抛出定义的异常
- 如果参数或者返回值是`map`、`set`、`list`实例，该实例不能被其他线程修改，否则会报协议错误或者超时(大家可以想下是为什么)

以上基本解释了Thrift的协议格式以及不同类型的序列化格式，还有我使用过程中遇到的一些坑，希望对大家理解Thrift有所帮助。