---
title: Scala的Parsers初探
tags: [scala,parser]
categories: 技术
--- 

### 00 缘起
在阅读Scrooge源码时发现其对Thrift的IDL解析竟然只用了一个类，不到500行代码，并且没用到JavaCC或者Antlr之类的库(lex/yacc)，非常小巧，仔细看源码，发现是继承Scala的RegexParsers实现的。

本文将简单介绍Scala的**scala-parser-combinators**。

### 01 介绍
**scala-parser-combinators**是Scala的一个module,[Github地址](https://github.com/scala/scala-parser-combinators)。

主要的类/接口有：

- Parser：一个或者一类输入的解析类，多个Parser之间可以互相组合
- Parsers：定义输入的类型，定义/组合/连接多个Parser
- Reader：输入源抽象，包括字符串/流等实现
- ParseResult：解析结果，成功/失败
- Positional：
- Position

该module提供了多种不同的Parsers，每一种分别有不同的特性，下图是Parsers的层级关系。
![Parsers类层级](/assets/img/parsers.png)

上图列出了module提供的Parsers，具体每个Parsers适合做什么，读者可以自己查看文档说明(英语渣就不瞎翻译了^_^)

**上面列了很多Parsers，是否可以直接使用呢？**答案是**否定**的。

- **Parsers**都是`trait`，真正工作的是**Parsers**内部的`Parser`类；
- **Parsers**只是实现了构建/连接/组合各种`Parser`之间规则的功能，并定义输入的类型；


**Parsers**提供了
|方法|说明|
|:-:|:-:|
|d|fd|



### 01 如何使用
Scala早期版本的SDK默认是集成此功能的，不过最近的版本把这个功能从SDK中抽出，独立成一个三方库。

博主使用maven构建项目，所以首先要依赖

```
<dependency>
  <groupId>org.scala-lang.modules</groupId>
  <artifactId>scala-parser-combinators_2.11</artifactId>
  <version>1.0.4</version>
</dependency>
```

