---
layout: post
title: mybatis初探
date: 2017-02-25 23:04:47
tags: [mybatis]
categories: 技术
---

最近升级到Spring4.x，DAO层也同步升级到最新版本的mybatis，写了一些代码生成插件，同时也学习下mybatis，不过代码注释确实不多，不过主流程还算能够看懂。

<!-- more -->
理解mybatis的调用流程，先确定核心的几个类：

- Configuration
- MapperProxy
- SqlSession
- Executor
- StatementHandler

通过这些类，组成了mybatis核心调用流程。

#### Configuration

mybatis启动会做加载全局配置文件、关联Mapper对应的sql、关联数据源等操作，结果会全部汇聚到Configuration中。

Configuration中的属性有很多，我比较关注的是：

- MapperRegistry管理所有的Mapper
- Envrionment管理数据源和事务
- TypeHandlerRegistry管理所有JDBC类型与Java类型之间的转换
- InterceptorChain管理拦截器
- MappedStatement执行sql映射关系缓存

还有很多其他属性，如缓存，但是说实话，这个我真的用不到。

Configuration实例会贯穿mybatis整个调用链中，可以理解为整个DAO层的Context上下文。

#### MapperProxy

mybatis和ibatis从表面上看最大的不同就是它只有接口(Mapper)，那么我们调用接口的实现在哪？

mybatis是对象SQL映射，调用一个SQL的过程可以抽象为3个主要部分：

- 解析参数
- 执行映射SQL
- 解析结果

既然每个实现的操作步骤是类似的，mybatis就用动态代理来生成代理实现类，而不用每个都生成一个实现，这个动态代理类就是MapperProxy。

MapperProxy看名字就很明显，是代理Mapper的，其本身实现了InvocationHandler接口，内部主要有3个属性：

- mapperInterface代理的mapper接口类，如`GiftMapper`
- methodCache是一个Map结构，key是Method，value中缓存的是MapperMethod对象
- sqlSession

MapperMethod内部依赖：

- MethodSignature管理包括参数类型、参数转换、返回值类型、返回值类型转换等属性;
- SqlCommand管理SQL的id和类型

#### SqlSession

一次事务可以有一个或者多个操作，通常在一个线程中依次执行，整个过程可以称之为一个会话，SqlSession我理解就是会话的抽象，会话结束后，也需要销毁。

SqlSession提供了多种对数据库操作的接口，如：

- selectOne
- selectMap
- selectList
- update
- insert
- delete
- commit
- rollback

基本涵盖了绝大多数数据库操作。

SqlSession的实现主要分2类：

- DefaultSqlSession
- SqlSessionManager和SqlSessionTemplate

DefaultSqlSession内部依赖Executor来执行具体的SQL，需要自己做事务操作，连接释放，异常处理等，但是这样对用户不是特别友好，所以就有了SqlSessionManager和SqlSessionTemplate。

SqlSessionManager用于独立使用mybatis，SqlSessionTemplate用于mybatis与spring集成。他们内部都依赖于一个DefaultSqlSession，并用的动态代理的方式，依赖SqlSessionInterceptor来做连接释放、异常处理、事务操作等。唯一的不同是SqlSessionManager对线程缓存了DefaultSqlSession的代理实例，而SqlSessionTemplate并没有缓存，而是根据事务声明情况分别采取不同的策略。

#### Executor

Executor并非最终执行SQL的抽象，其内部会构造StatementHandler实例，并最终由StatementHandler来执行SQL,抽象出Executor层mybatis做了缓存、批处理、Statement重用以及最基本的实现。

- CachingExecutor
- BatchExecutor
- ReuseExecutor
- SimpleExecutor

##### CachingExecutor

结合一些分布式缓存来提高查询效率，降低数据库压力。不过一般互联网会使用到mybatis缓存的并不多见，单体的系统可能会用到，个人认为作为一个可选模块可能更好，这里不多介绍。

##### BatchExecutor

继承自抽象类BaseExecutor，一些数据迁移、数据交换的场景可能会用到，能够极大的提升效率。

##### ReuseExecutor

继承自抽象类BaseExecutor，内部使用一个Map来缓存Statement，在一个事务中尽量重用Statement

##### SimpleExecutor

最基本的Executor实现，继承自抽象类BaseExecutor，默认使用此Executor。

#### StatementHandler

最终依赖JDBC执行SQL的抽象，分为：

- CallableStatementHandler
- PreparedStatementHandler
- SimpleStatementHandler
- RoutingStatementHandler

前3个对应JDBC的CallableStatement、PreparedStatement和Statement。

RoutingStatementHandler采用委托模式，根据实际请求，委托对应的StatementHandler来执行Statement。

StatementHandler依赖ParameterHandler来处理参数，依赖ResultSetHandler来处理SQL执行结果，依赖TypeHandler来处理JDBC Type与Java Type之间的转换。

#### 总结

调用流程的关系如下图:

![image](/assets/img/mybatis001.png)

理解这几个抽象，应该就能大概的了解mybatis的整个调用流程了。


