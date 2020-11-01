---
layout: post
title:  "Scala中的函数"
date:   2020-7-27
categories: Scala
keywords: Scala, Function
mathjax: false
author: wzx
---

Scala是函数式编程语言，下面记录一些比较困扰的函数概念。




## 函数与方法

在Scala中函数是一等公民，方法可不是。两者语义上比较相似，但是还是有很大的区别。**方法是类的一部分，而函数是一个对象可以赋值给一个变量。**

```scala
object Test {
  // 方法
  def m1(msg: String) = println(msg)
  // 没有参数时，参数列表可以不写
  def m2 = println("method")

  // 函数
  val f1 = (msg: String) => println(msg)
  val f2 = () => println("function")

  def high(f: (String) => Unit): Unit = {
    println(f)
  }

  // 方法调用
  m1("method")
  m2
  // 函数调用
  f1("function")
  // 调用时必须有参数列表
  f2()

  // 传递函数
  high(f1)
  // m1 _ 将m转化为函数
  high(m1 _)
  // 这种情况下，默认方法会被转化成函数
  high(m1)
}
```



## 传值与传名调用

- 传值调用（call-by-value）：先计算参数表达式的值，再应用到函数内部；
- 传名调用（call-by-name）：将未计算的参数表达式直接应用到函数内部

```scala
object Test {

  def time() = {
    println("in time")
    System.nanoTime
  }

  // 函数的传名调用
  def delayed1(t: => Long) = {
    println("in the method")
    println(t)
  }

  // 传值调用
  def delayed2(t: Long) = {
    println("in the method")
    println(t)
  }

  // 直接传递函数
  def delayed3(t: () => Long) = {
    println("in the method")
    println(t())
  }

  /**
   * in the method
   * in time
   * 616895450693991
   * in time
   * in the method
   * 616895450765043
   * in the method
   * in time
   * 616895451213135
   * in time
   * in the method
   * 616895451250756
   */
  def main(args: Array[String]) {
    delayed1(time())
    delayed2(time())
    delayed3(time _)

    // 直接传递的传名调用
    delayed2{
      println("in time")
      System.nanoTime
    }

  }
}
```

## 偏函数

**偏函数是一种特殊的一元函数，它并不会接受符合参数类型的所有可能值，而是只接受特定的值**。也就是说偏函数只接受其参数定义域的一个子集，而对于这个子集之外的参数则抛出运行时异常，**与模式匹配混合使用**。

```scala
object Test {

  // 第一个是输入值类型，第二个是返回值类型
  val positive: PartialFunction[Int, Int] = {
    case x if x >= 0 => x
  }

  val odd: PartialFunction[Int, Boolean] = {
    case x if x % 2 == 1 => true
  }

  val even: PartialFunction[Int, Boolean] = {
    case x if x % 2 == 0 => true
  }

  // andThen orElse组合仍返回偏函数
  val evenCheck: PartialFunction[Int, Boolean] = positive andThen even
  val oddCheck: PartialFunction[Int, Boolean] = positive andThen odd

  evenCheck.isDefinedAt(-1)   // false
  evenCheck(-1) // throw exception
}
```



## 部分应用函数

**调用函数的过程也叫做将函数应用到参数**，当只传入部分参数时，Scala并不会报错，而是简单地应用了这些参数，并**返回一个接受剩余参数的新函数**。

```scala
object Test {

  // (Int, Int, Double) => Double
  val add = (a: Int, b: Int, c: Double) => a + b + c
  // _表示剩余参数的占位符
  // (Int, Double) => Double
  val add2 = add(_: Int, 2, _: Double)
  // 42.0
  add2(20, 20.0)
}
```



## 柯里化

柯里化和部分应用函数都是对函数进行降元，但柯里化使得可以定义**多个参数列表**，当使用较少的参数列表调用多参数列表的方法时，会产生一个新的函数，该函数接收剩余的参数列表作为其参数。**柯里化必须按照定义的顺序调用**。

```scala
object Test {

  def addMethod(a: Int)(b: Int) = a + b
  val addFunc = (a: Int, b: Int) => a + b

  // Int => Int => Int
  val curried1 = addMethod _
  // Int => Int => Int
  val curried2: (Int) => (Int) => Int = addFunc.curried

  // Int => Int
  val res = curried1(2)
  // 3
  res(1)
}
```



## REFERENCE

1. [scala文档](https://docs.scala-lang.org/zh-cn/tour)
2. [快学Scala](https://book.douban.com/subject/19971952/)
3. [ronoob](https://www.runoob.com/scala/scala-tutorial.html)
4. [【译】Scala的偏函数和部分应用函数](https://zhuanlan.zhihu.com/p/33165576)
