---
layout: post
title:  "Scala中的隐式变换"
date:   2020-8-3
categories: Scala
keywords: Scala, 隐式变换
mathjax: false
author: wzx
---

总结scala隐式变换中的容易混淆的概念。





## 隐式转换

以`implicit`关键字声明的**单个参数**的函数即为隐式转换函数

### 引入规则
- **位于源或目标类型的伴生对象中的隐式函数**
- **当前作用域中可以以单个标识符指代的隐式函数**，即需要导入到具体的函数
    - `import spark.implicit._`，可以隐式转换
    - `import spark.implicit`，只能`implicit.xx`显示调用
    - `import spark.implicit.{xx => _, _}`，导入出`xx`外的所有隐式函数

### 转换规则
- **表达式类型与预期类型不同**
- **对象访问不存在成员**
- 对象调用方法时，**传入参数与参数声明类型不匹配**
- **不适用隐式转换的前提下可以通过编译**，就不会使用隐式转换
- **不能同时执行多个隐式转换**
- **隐式转换不存在二义性**。不能两个隐式转换同时满足

```scala
case class Fraction(numerator: Int, denominator: Int)

class FractionEnhanced(n: Fraction) {
  def toInt: Int = n.numerator / n.numerator
  // 定义在目标类型中
  implicit def Fraction2Enhanced(n: Fraction): FractionEnhanced = new FractionEnhanced(n)
}

object Fraction {
  // 定义在源类型中
  implicit def Fraction2Int(n: Fraction): Int = n.numerator / n.numerator
}

object test {

  def main(args: Array[String]): Unit = {
    // 类型与预期不同
    // 3 * Fraction2Int(Fraction(1, 2))
    3 * Fraction(1, 2)

    // 调用不存在成员
    // Fraction2Int(Fraction(1, 2)).toInt
    Fraction(1, 2).toInt
  }
}
```

### 利用隐式参数进行隐式转换

```scala
def smaller[T](a: T, b: T)(implicit order: T => Ordered[T]): T = {
  // if (order(a) < b) a else b
  if (a < b) a else b
}
```

## 隐式参数

方法可以具有**隐式参数列表**，由参数列表开头的`implicit`关键字标记。

如果参数列表中的参数没有传递隐式参数列表中的参数，将从下两个地方寻找**可用单个标识符指定**的**满足类型要求**的替代值

- 当前**作用域中**的val和def
  ```scala
  object test {
    implicit val gender: String = "male"

    def main(args: Array[String]): Unit = {
      implicit val age: Int = 10

      def person(name: String)(implicit gender: String, age: Int): Unit = println(s"age: $age; name: $name; gender: $gender")

      // 在作用域内没有歧义(类型)时，可以自动传入隐式参数
      // country: 10; name: wzx; gender: male
      person("wzx")
    }
  }
  ```
- 目标类型的伴生对象，包含其子类
  ```scala
  trait Calculate[T] {
      def add(x: T, y: T): T
  }

  implicit object IntCal extends Calculate[Int] {
      def add(x: Int, y: Int): Int = x + y
  }

  implicit object ListCal extends Calculate[List[Int]] {
      def add(x: List[Int], y: List[Int]): List[Int] = x ::: y
  }

  def implicitObjMethod[T](x: T, y: T)(implicit cal: Calculate[T]): Unit = {
      println(x + " + " + y + " = " + cal.add(x, y))
  }

  implicitObjMethod(1, 2) // 1 + 2 = 3
  implicitObjMethod(List(1, 2), List(3, 4)) // List(1, 2) + List(3, 4) = List(1, 2, 3, 4)
  ```


获取隐式值
```scala
// implicit object Int extends IntOrdering
// trait IntOrdering extends Ordering[Int]

// 从冥界召唤出隐式值Ordering[Int]
// val ord = implicitly[Ordering[Int]](Int)
val ord = implicitly[Ordering[Int]]
```

## 隐式类

在对应的作用域内，隐式类的主构造函数可用于隐式转换。

- 隐式类只能在别的trait, class, object内部定义
- 其构造函数只能有一个非隐式参数
- 在同一作用域内，不能有任何方法、成员或对象与隐式类同名

```scala
scala> object Helpers {
     |   implicit class IntWithTimes(x: Int) {
     |     def times[A](f: => A): Unit = {
     |       def loop(current: Int): Unit =
     |         if(current > 0) {
     |           f
     |           loop(current - 1)
     |         }
     |       loop(x)
     |     }
     |   }
     | }
defined object Helpers

scala> import Helpers._
import Helpers._

scala> 5 times println("HI")
HI
HI
HI
HI
HI
```



## REFERENCE

1. [scala文档](https://docs.scala-lang.org/zh-cn/tour)
2. [快学Scala](https://book.douban.com/subject/19971952/)
