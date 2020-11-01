---
layout: post
title:  "Scala中的泛型"
date:   2020-8-3
categories: Scala
keywords: Scala, 泛型
mathjax: false
author: wzx
---

总结scala泛型中的容易混淆的概念。





## 类型变量界定
- `T >: A`：下界
- `T <: A`：上界

## 视图界定
`T <% Ordered[T]`：T可以隐式转换为`Ordered[T]`

## 上下文界定
`T:M`：`M`是泛型类时，要求**作用域存在一个隐式值`M[T]`**

```scala
// 实例化时，要求作用域存在一个隐式值`Ordering[T]`
class Pair[T: Ordering](val first: T, val second: T) {
  def smaller(implicit ord: Ordering[T]): T = if (ord.compare(first, second) < 0) first else second
}

object test {

  def main(args: Array[String]): Unit = {

    // implicit object Int extends IntOrdering
    // trait IntOrdering extends Ordering[Int]

    // val pair = new Pair[Int](1, 2)(Int)
    val pair = new Pair(1, 2)
    // pair.smaller(Int)
    pair.smaller
  }
}
```

## ClassTag上下文界定
防止泛型擦除

```scala
object test {
  // ClassTag，所以等同于存在 ClassTag[T]
  def toArray1[T: ClassTag](value: List[T]) = Array[T](value: _*)

  // 编译错误，因为泛型擦除，List[T]会擦除T类型
  def toArray2[T](value: List[T]) = Array[T](value: _*)

  def main(args: Array[String]): Unit = {
    // toArray1(List(1, 2, 3))(ClassTag[T])
    // int[]
    println(toArray1(List(1, 2, 3)).getClass.getSimpleName)
  }
}
```

## 类型约束
- `T =:= u`：测试`T`是否为`U`
- `T <:< u`：测试`T`是否为`U`的子类

```scala
class Pair1[T] {
  def test(i: T)(implicit ev: T <:< java.io.Serializable) = print("Serializable")
}

class Pair2[T <: java.io.Serializable] {
  def test(i: T) = print("Serializable")
}

object test {

  def main(args: Array[String]): Unit = {
    val pair1 = new Pair1[Int]
    // 调用时出错，因为找不到 Int <::< java.io.Serializable
    pair1.test(1)

    val pair2 = new Pair1[String]
    // sealed abstract class <:<[-From, +To] extends (From => To) with Serializable
    // implicit def $conforms[A]: A <:< A = singleton_<:<.asInstanceOf[A <:< A]
    pair2.test("1")

    // 编译错误
    val pair3 = new Pair2[Int]
  }
}
```

## 协变, 逆变
> covariant, contravariant

```scala
class Foo[+A] // A covariant class
class Bar[-A] // A contravariant class
class Baz[A]  // An invariant class
```
- 如果B是A的子类，定义`List[+A]`，`List[B]`是`List[A]`的子类
- 如果B是A的子类，定义`List[-A]`，`List[A]`是`List[B]`的子类

## REFERENCE

1. [scala文档](https://docs.scala-lang.org/zh-cn/tour)
2. [快学Scala](https://book.douban.com/subject/19971952/)
