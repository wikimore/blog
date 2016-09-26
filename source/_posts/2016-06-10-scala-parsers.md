---
layout: post
date: 2016-06-10 16:58:14
title: Scala的Parsers初探
tags: [scala,parser]
categories: 技术
---

**scala-parser-combinators**是Scala的一个module，[Github地址](https://github.com/scala/scala-parser-combinators)，使用者可以通过定义多个Parser，并组合起来，完成想要的工作。
<!-- more -->

主要的类/接口有：

- Parser：一个或者一类输入的解析类，多个Parser之间可以互相组合
- Parsers：定义输入的类型，定义/组合/连接多个Parser
- Reader：输入源抽象，包括字符串/流等实现
- ParseResult：解析结果，成功/失败/错误等
- Positional：定位源的位置
- Position：表示位置


Parser有几个主要的方法，通过这些方法，可以完成多个parser的组合。

- `p ~ q`：p成功并且余下的输入对q也成功，返回`p的结果~q的结果`
- `p ~> q`：p成功并切后面输入对q也成功，返回`q的结果`
- `p <~ q`：p成功并切后面输入对q也成功，返回`p的结果`
- `p ~! q`：p成功并且余下的输入对q也成功，返回`p的结果~q的结果`，和`~`的区别是如果失败，不会回溯，直接就报错了
- `p | q`：p成功或者q成功，如果p失败，会回溯匹配q，否则直接返回`p的结果`
- `p ||| q`：p成功或者q成功，如果p失败，会回溯匹配q，即使p成功，也会匹配q，如果p和q都成功，则返回p和q的结果中最长的那个
- `p ^^ f`：如果p成功，返回函数f处理后的p的结果
- `p ^^^ v`：如果p成功，丢弃p的结果，直接返回v的结果
- `p ^? f`：如果p匹配成功，并且`f`的isDefinedAt执行`p`的结果返回true，返回函数f处理后的p的结果
- `p *`：重复的使用p来匹配，直到匹配失败，返回一个匹配成功的结果的数组，可以都不成功
- `p +`：重复的使用p来匹配，直到匹配失败，返回一个匹配成功的结果的数组，至少必须成功一次
- `p ?`：可选的匹配，匹配成功返回`Some(x)`，失败返回`None`

Parsers还有一些常用的方法：

- `rep(p)`：重复的使用p来匹配，直到匹配失败，返回一个匹配成功的结果的数组，可以都不成功
- `rep1(p)`：重复的使用p来匹配，直到匹配失败，返回一个匹配成功的结果的数组，至少必须成功一次
- `rep1(p,q)`：先使用p匹配一次，这一次必须成功，然后重复的使用q来匹配，直到q匹配失败，返回一个匹配成功的结果的数组
- `repN(n, p)`：重复的使用p来匹配，返回一个匹配成功的结果的数组，必须刚好匹配成功N次
- `opt(p)`：可选的匹配，匹配成功返回`Some(x)`，失败返回`None`

下面是一个计算器(整数的加减乘除)的例子，通过多种不同的parser的组合，共同完成表达式的解析。

```
import scala.util.parsing.combinator.RegexParsers
import scala.util.parsing.combinator.ImplicitConversions

object Calculator extends RegexParsers with ImplicitConversions {
  lazy val number: Parser[Int] = """-?\d+""".r ^^ { _.mkString.toInt } // 匹配整数
  lazy val expr0: Parser[Int] = number | '(' ~> expr2 <~ ')'
  lazy val expr1: Parser[Int] = expr0 ~ rep('*' ~ expr0 | '/' ~ expr0) ^^ {
    case (number ~ list) => list.foldLeft(number) {
      case (x, ('*' ~ y)) => x * y
      case (x, ('/' ~ y)) => x / y
    }
  }
  lazy val expr2: Parser[Int] = expr1 ~ rep('+' ~ expr1 | '-' ~ expr1) ^^ {
    case (number ~ list) => list.foldLeft(number) {
      case (x, ('+' ~ y)) => x + y
      case (x, ('-' ~ y)) => x - y
    }
  }

  def parse(in: String): Int = {
    val result = parseAll(expr2, in)
    result.get
  }

  def main(args: Array[String]): Unit = {
    println(Calculator.parse("1+(2*2)+(-3)+(4+6/3)*5"))
  }
}
```

