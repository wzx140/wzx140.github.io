---
layout: post
title:  "Scala-模式匹配"
date:   2020-6-19
categories: Scala
keywords: Scala CaseMatching
mathjax: false
author: wzx
---

Scala中的模式匹配功能很强大，它是Java中的`switch`语句的升级版，同样可以用于替代一系列的 if/else 语句



## 语法

Scala的模式匹配是表达式，而不是语句，与Java中 switch 语句类似，注意必须包含一条case语句，如果所有的case语句都不匹配，那么会抛出`InvocationTargetException`异常

```scala
import scala.util.Random

val x: Int = Random.nextInt(10)

x match {
  case 0 => "zero"
  case 1 => "one"
  case 2 => "two"
  // default
  case _ => "other"
}
```

## 常量匹配

用于匹配任何类型的常量

```scala
x match {
  case 1 => "One"
  case "2" => "Two"
  case true => "True"
  case null => "null value"
  case Nil => "empty list"
  case _ => "other value"
}
```

## 变量匹配

如果case语句中包含变量名，则匹配的表达式会赋值给这个变量，**利用守卫替代if语句**。

特别注意，**模式匹配作用域之外的变量不会被case关键词之后的变量关联**，除非使用**首字母大写的变量名**或者使用**反引号包裹**

```scala
val X = "1"
val x = "3"

def getMatch(x: String): String = {
  x match {
    // 小写模式将被当做变量
    // 守卫
    case x if x == "2" => "inner 2"
    // 大写字母开头
    case X => "outer 1"
    // 反引号包裹
    case `x` => "outer 3"
    // 类型匹配
    case x: String => "other value:" + x
  }
}

// outer 1
// inner 2
// outer 3
println(getMatch("1"))
println(getMatch("2"))
println(getMatch("3"))
```

## 类型匹配

模式匹配可以匹配类型。**注意匹配发生在运行期，泛型信息会被擦除而不起作用。**

```scala
x match {
  case s: String => s"the string length is: ${s.length}"
  // 不要使用 Map[Int, String] 这样不起作用
  case m: Map[_, _] => "the map size is:" + m.size
  case _: Int | _: Double => s"the number is: $x"
  case _ => "unexpected value"
}
```



## 元组序列匹配

可以轻松地匹配到元组或者序列的不同部分

```scala
// 元组
x match {
  case (first, _, _) => first
  case _ => "Something else"
}

// 序列
x match {
  case 0 :: Nil => "只含0"
  case x :: y :: Nil => s"只包含两个元素 $x $y"
  case 0 :: tail => "0开头"
  case List(_*) :+ 0 => "0结尾"
  case _ => "Other seq"
}
```

## 提取器

在模式匹配中可以结合`unapply`和`unapplySeq`方法用于提取对象中的值

```scala
val pattern = "([0-9]+) ([a-z]+)".r
arr match {
    case Array(0, x) => x
    case pattern(num, alphabet) => s"num: $num alphabet: $alphabet"
}
```

## case class匹配

案例类经过优化一般用于模式匹配，案例类具有以下约束

- 构造参数默认`val`
- 默认**按值比较**
- 实例化不使用`new`，默认通过`apply()`方法实例化

```scala
abstract class Notification
case class Email(sender: String, title: String, body: String) extends Notification
case class SMS(caller: String, message: String) extends Notification
case class VoiceRecording(contactName: String, link: String) extends Notification

def showImportantNotification(notification: Notification, importantPeopleInfo: Seq[String]): String = {
  notification match {
    case Email(sender, _, _) if importantPeopleInfo.contains(sender) =>
      "You got an email from special someone!"
    case SMS(number, _) if importantPeopleInfo.contains(number) =>
      "You got an SMS from special someone!"
    // 需要使用case class本身时，使用 @ 符号
    case p @ SMS(number, _) if p.message == null =>
      "You got none SMS!"
    case other =>
			throw new Exception("no match")
  }
}
```

## 异常处理

```scala
catch {
  case e: IllegalArgumentException => println("illegal arg. exception");
  case e: IllegalStateException    => println("illegal state exception");
  case e: IOException              => println("IO exception");
}
```

## REFERENCE

[1] [[Why does pattern matching in Scala not work with variables?](https://stackoverflow.com/questions/7078022/why-does-pattern-matching-in-scala-not-work-with-variables)

[2] [Scala doc](https://docs.scala-lang.org/zh-cn/tour/pattern-matching.html)

[3] [快学Scala](https://book.douban.com/subject/19971952/)
