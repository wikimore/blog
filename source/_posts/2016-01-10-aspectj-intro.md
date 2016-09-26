---
layout: post
date: 2016-01-10 16:31:29
title: AspectJ浅析
tags: aspectj
categories: 技术
---

### 00 概述

AspectJ是一个面向切面的框架：

- 定义了一套面向切面的语法
- 一个扩展了javac的编译器，编译java或者aj的源代码，生成遵守Java虚拟机规范的字节码
- 支持编译器织入和加载时织入
<!-- more -->
### 01 编译期织入(Compile Time Weaver)

编译期织入是指在编译期，使用AspectJ的编译器(ajc)编译.aj以及.java文件，根据配置，对.java源码织入动态逻辑。

一般使用Eclipse的AJDT工具或者maven的aspectj插件来自动编译。

动手写一个使用maven的aspectj插件来编译的例子：

先贴一个App类，有一个main方法，内部分别调用log4j的log和自己写的一个log方法。

```
package io.github.wikimore.ajtest;

import org.apache.log4j.Logger;

public class App {
    private static final Logger LOG = Logger.getLogger("AspectJ");
    public static void main(String[] args) {
        LOG.error("message");
        Log.log("lllllll");
    }
}
```

自己写的Log很简单，就是打印到stdout中

```
package io.github.wikimore.ajtest;

public class Log {
    public static void log(String s) {
        System.out.println("log ---> " + s);
    }
}
```

main方法的输出应该是

```
ERROR (AspectJ:13) - message
log ---> lllllll
```

> log4j配置不同输出不完全一样

以上和aspectj还没有任何关系，下面我们想在2个方法的前面后面加一些输出：

#### 编写aj源文件：定义AOP的类、方法以及织入的代码

Log4jAspect.aj为织入log4j的代码

```
package io.github.wikimore.ajtest.aspectj.config;

public aspect Log4jAspect {

    pointcut beforeExecute():call(* org.apache.log4j.Appender.doAppend(*));

    Object around() throws Throwable:beforeExecute() {
        System.out.println("before log4j log");
        Throwable throwable = null;
        Object retVal = null;
        try{
            retVal = proceed();
        } catch (Throwable t) {
            throwable = t;
        }
        if(throwable != null ) {
        	    System.out.println("error");
            throw throwable;
        }
			System.out.println("after log4j log");
   		return retVal;
    }

}
```

LogAspect.aj为织入Log的代码

```
package io.github.wikimore.ajtest.aspectj.config;

public aspect LogAspect {

    pointcut beforeExecute():call(* io.github.wikimore.ajtest.Log.log(*));

    Object around() throws RuntimeException:beforeExecute() {
        System.out.println("before log");
        Throwable throwable = null;
        Object retVal = null;
        try{
            retVal = proceed();
        } catch (Throwable t) {
            throwable = t;
        }
        if(throwable != null ) {
        	System.out.println("error");
            throw new RuntimeException(throwable);
        }
		   System.out.println("after log");
   		return retVal;
    }

}
```

#### 编写aop的xml配置文件：明确哪些aj源文件会被编译(代码织入)，该文件放到META-INF文件夹下，命名aop.xml

```
<?xml version="1.0"?>
<aspectj>
	<weaver options="-verbose -showWeaveInfo">
		<!--指定weave范围 -->
		<include within="org.apache.log4j..*" />
		<include within="io.github.wikimore.ajtest..*" />
	</weaver>
	<aspects>
		<aspect name="io.github.wikimore.ajtest.aspectj.config.Log4jAspect" />
		<aspect name="io.github.wikimore.ajtest.aspectj.config.LogAspect" />
	</aspects>
</aspectj>
```

#### maven增加aspectj的编译插件，需要用ajc来编译(代替javac)

```
<plugin>
	<groupId>org.codehaus.mojo</groupId>
	<artifactId>aspectj-maven-plugin</artifactId>
	<version>1.5</version>
	<configuration>
		<complianceLevel>1.6</complianceLevel>
		<includes>
			<include>**/*.java</include>
			<include>**/*.aj</include>
		</includes>
		<!-- aj文件夹的路径 -->
		<aspectDirectory>src/main/java/io/github/wikimore/ajtest/aspectj.config</aspectDirectory>
		<XaddSerialVersionUID>true</XaddSerialVersionUID>
		<showWeaveInfo>true</showWeaveInfo>
	</configuration>
	<executions>
		<execution>
			<id>compile_with_aspectj</id>
			<goals>
				<goal>compile</goal>
			</goals>
		</execution>
		<execution>
			<id>test-compile_with_aspectj</id>
			<goals>
				<goal>test-compile</goal>
			</goals>
		</execution>
	</executions>
	<dependencies>
		<dependency>
			<groupId>org.aspectj</groupId>
			<artifactId>aspectjtools</artifactId>
			<version>1.7.3</version>
		</dependency>
	</dependencies>
</plugin>
```

执行`mvn clean compile`，编译后运行App的main方法发现输出如下：

```
ERROR (AspectJ:13) - message
before log
log ---> lllllll
after log
```

我们自己编写的类已经被aop了，但是log4j并没有，很疑惑？

接下来执行javap或者用反编译工具看看生成的class是什么样的。

下面是javap打印出的LogAspect.class和Log4jAspect.class

```
Compiled from "LogAspect.aj"
public class io.github.wikimore.ajtest.aspectj.config.LogAspect extends java.lang.Object{
    public static final io.github.wikimore.ajtest.aspectj.config.LogAspect ajc$perSingletonInstance;
    static {};
    public io.github.wikimore.ajtest.aspectj.config.LogAspect();
    void ajc$pointcut$$beforeExecute$5f();
    public java.lang.Object ajc$around$io_github_wikimore_ajtest_aspectj_config_LogAspect$1$c5f6d277(org.aspectj.runtime.internal.AroundClosure)       throws java.lang.RuntimeException;
    static java.lang.Object ajc$around$io_github_wikimore_ajtest_aspectj_config_LogAspect$1$c5f6d277proceed(org.aspectj.runtime.internal.AroundClosure)       throws java.lang.Throwable;
    public static io.github.wikimore.ajtest.aspectj.config.LogAspect aspectOf();
    public static boolean hasAspect();
}

Compiled from "Log4jAspect.aj"
public class io.github.wikimore.ajtest.aspectj.config.Log4jAspect {
  public static final io.github.wikimore.ajtest.aspectj.config.Log4jAspect ajc$perSingletonInstance;
  static {};
  public io.github.wikimore.ajtest.aspectj.config.Log4jAspect();
  void ajc$pointcut$$beforeExecute$61();
  public java.lang.Object ajc$around$io_github_wikimore_ajtest_aspectj_config_Log4jAspect$1$c5f6d277(org.aspectj.runtime.internal.AroundClosure) throws java.lang.Throwable;
  static java.lang.Object ajc$around$io_github_wikimore_ajtest_aspectj_config_Log4jAspect$1$c5f6d277proceed(org.aspectj.runtime.internal.AroundClosure) throws java.lang.Throwable;
  public static io.github.wikimore.ajtest.aspectj.config.Log4jAspect aspectOf();
  public static boolean hasAspect();
}
```

aj文件编译后生成了很多类似`ajc$around$`的方法，但是为什么log4j的没有被AOP呢？

再用jd-gui查看App.class和Log.class

```
public static void main(String[] args) {
	LOG.error("message");
	String str = "lllllll"; log_aroundBody1$advice(str, LogAspect.aspectOf(), null);
}
```

```
public class Log
{
  public static void log(String s)
  {
    System.out.println("log ---> " + s);
  }
}
```

- App中的`Log.log("lllllll");`方法被`替换`了！！！但是`LOG.error("message");`并没有被替换
- Log本身代码并没有变化

> log_aroundBody1$advice这个方法在哪里我也没找到。。

从这个我们大致可以推测出：

- aj编译器不能在编译时对已经编译好的代码进行织入(log4j)
- aj编译时织入是在调用处，而非代码实现处，可以理解为aspectj用一个方法`log_aroundBody1$advice`包住了要aop的方法`log`，所有调用`log`的地方改为调用`log_aroundBody1$advice`

以上就是aspectj的CTW，如果要AOP log4j的方法，就需要LTW了。

### 02 加载期织入(Loading Time Weaver)

加载期织入提供一中更加灵活的方式，在类被虚拟机加载的时候，动态的织入代码到目标类中。

LTW依赖于java的agent，不了解的可以先参考[Oracle文档](https://docs.oracle.com/javase/7/docs/api/java/lang/instrument/package-summary.html)、[JSR-163](https://jcp.org/en/jsr/detail?id=163)，下面也会有说到，现在市面上很多APM厂商监控Java就是基于agent。

上面的例子我们需要在执行时加入一些参数：

```
// ${path}替换成你的路径
-javaagent:/${path}/aspectjweaver-1.7.3.jar
```

输入如下：

```
[AppClassLoader@40affc70] info AspectJ Weaver Version 1.7.3 built on Thursday Jun 13, 2013 at 19:41:31 GMT
[AppClassLoader@40affc70] info register classloader sun.misc.Launcher$AppClassLoader@40affc70
[AppClassLoader@40affc70] info using configuration /Users/nali/dev/frmworkspace/ajtest/target/classes/META-INF/aop.xml
[AppClassLoader@40affc70] info register aspect io.github.wikimore.ajtest.aspectj.config.Log4jAspect
[AppClassLoader@40affc70] info register aspect io.github.wikimore.ajtest.aspectj.config.LogAspect
[AppClassLoader@40affc70] info processing reweavable type io.github.wikimore.ajtest.App: io/github/wikimore/ajtest/App.java
[AppClassLoader@40affc70] info successfully verified type io.github.wikimore.ajtest.aspectj.config.LogAspect exists.  Originates from io/github/wikimore/ajtest/aspectj/config/Users/nali/dev/frmworkspace/ajtest/src/main/java/io/github/wikimore/ajtest/aspectj/config/LogAspect.aj
[AppClassLoader@40affc70] weaveinfo Join point 'method-call(void io.github.wikimore.ajtest.Log.log(java.lang.String))' in Type 'io.github.wikimore.ajtest.App' (App.java:14) advised by around advice from 'io.github.wikimore.ajtest.aspectj.config.LogAspect' (LogAspect.aj:7)
[AppClassLoader@40affc70] error at org/apache/log4j/helpers/AppenderAttachableImpl.java:66::0 can't throw checked exception 'java.lang.Throwable' at this join point 'method-call(void org.apache.log4j.Appender.doAppend(org.apache.log4j.spi.LoggingEvent))'
[AppClassLoader@40affc70] error at io/github/wikimore/ajtest/aspectj/config/Users/nali/dev/frmworkspace/ajtest/src/main/java/io/github/wikimore/ajtest/aspectj/config/Log4jAspect.aj:7::0 can't throw checked exception 'java.lang.Throwable' at this join point 'method-call(void org.apache.log4j.Appender.doAppend(org.apache.log4j.spi.LoggingEvent))'
[AppClassLoader@40affc70] weaveinfo Join point 'method-call(void org.apache.log4j.Appender.doAppend(org.apache.log4j.spi.LoggingEvent))' in Type 'org.apache.log4j.helpers.AppenderAttachableImpl' (AppenderAttachableImpl.java:66) advised by around advice from 'io.github.wikimore.ajtest.aspectj.config.Log4jAspect' (Log4jAspect.aj:7)
[AppClassLoader@40affc70] info processing reweavable type io.github.wikimore.ajtest.aspectj.config.Log4jAspect: io/github/wikimore/ajtest/aspectj/config/Log4jAspect.aj
before log4j log
ERROR (AspectJ:13) - message
after log4j log
[AppClassLoader@40affc70] info processing reweavable type io.github.wikimore.ajtest.aspectj.config.LogAspect: io/github/wikimore/ajtest/aspectj/config/LogAspect.aj
before log
log ---> lllllll
after log
```

可以发现aspectj在执行前进行了代码织入操作。

#### 021 如何实现LTW？

JSR-163提供了preMain入口，可以再main方法被执行前，执行逻辑，需要在META-INF/MENIFEST.MF增加类似如下代码

```
Premain-Class: org.aspectj.weaver.loadtime.Agent
```
这里的Agent就是aspectjweaver的LTW入口类，该类必须有一个`premain`方法

```
 public static void premain(String options, Instrumentation instrumentation) {
        /* Handle duplicate agents */
        if (s_instrumentation != null) {
            return;
        }
        s_instrumentation = instrumentation;
        s_instrumentation.addTransformer(s_transformer);
    }
```

这里的Instrumentation可以增加一个ClassFileTransformer实现(具体说明可以查看Javadoc)。

ClassFileTransformer提供transform方法，该方法的官方解释如下：

> The implementation of this method may transform the supplied class file and return a new replacement class file.

发现bytecode在这里是可以被替换的，具体的替换逻辑还在依赖的类里。

ClassFileTransformer依赖aspectj提供了一个ClassPreProcessor接口，该接口最主要的方法是preProcess，具体实现类是Aj，但是Aj类并没有自己实现weave逻辑，而是依赖WeavingAdaptor的子类ClassLoaderWeavingAdaptor来实现。

在初始化ClassLoaderWeavingAdaptor时，会顺序加载META-INF/aop.xml|META-INF/aop-ajc.xml|org/aspectj/aop.xml，获取拦截的定义，具体在initialize方法中，通过DefaultWeavingContext来加载。

WeavingAdaptor主要方法是weaveClass，从名字上就能看出来，这个类也不是真正执行weave的，WeavingAdaptor依赖BcelWeaver，BCEL是什么？官方解释如下：

>The Byte Code Engineering Library (Apache Commons BCEL™) is intended to give users a convenient way to analyze, create, and manipulate (binary) Java class files (those ending with .class). Classes are represented by objects which contain all the symbolic information of the given class: methods, fields and byte code instructions, in particular.

原来这家伙是操作bytecode的，BcelWeaver依赖BcelClassWeaver来织入代码，内部应该有一个编译器来完成真正的织入工作。

### 03 总结

AspectJ可以更灵活的编写Java代码，使用LTW功能可以拦截第三方库，做一些扩展功能。

