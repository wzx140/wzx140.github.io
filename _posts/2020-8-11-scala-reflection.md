---
layout: post
title:  "Scala中的反射"
date:   2020-8-11
categories: Scala
keywords: Scala, 反射
mathjax: false
author: wzx
---

总结scala反射中的基本概念。





## 编译时反射
利用元编程的形式可以在编译期间修改自己，主要通过 macros 的形式。

## 运行时反射
在运行时给定一个类型或者对象的实例，运行时反射可以做到以下几点

### `Universes`
反射环境的区别主要在于是运行时反射还是编译时反射，这个差异封装在`Universe`中。所以`Universes`是反射的接入点，提供了用于反射中使用的所有主要概念的接口，例如`Types`, `Trees`和`Annotations`

```scala
import scala.reflect.runtime.{universe => ru}    // for runtime reflection
import scala.reflect.macros.Universe       // for compile-time reflection
```

### `Mirrors`
scala反射的核心部分。反射提供的所有信息都可以通过`Mirror`访问，而且可以在通过`Mirror`进行反射操作。根据要获得的信息类型或要采取的反射操作，必须使用不同的`Mirror`。

`Classloader mirrors`用于获取类型，并可以构造特定的`invoker mirrors`实现反射调用，用于访问方法，构造函数，字段等

- `Classloader mirror`：通过类加载器构造。
    - `Class mirror`：用于创建构造器的`invoker mirrors`
    - `Instance mirror`：用于创建实例的构造方法，字段，内部类和内部对象的`invoke mirrors`
        - `Method mirror`：用于调用实例方法，scala中仅有实例方法
        - `Field mirror`：用于获得或者改变实例字段，scala中仅有实例字段
    - `Module mirror`：单例对象的实例

```scala
scala> class class1 { val f = 2; def m = 2 }
defined class class1

scala> case class class2(x: Int)
defined class class2

scala> object object1 { def x = 2 }
defined object object1

// runtime reflect
scala> val ru = scala.reflect.runtime.universe
ru: scala.reflect.api.JavaUniverse = ...

// Classloader mirror
scala> val m = ru.runtimeMirror(getClass.getClassLoader)
m: ru.Mirror = ...

// Instance mirror
scala> val im = m.reflect(new class1)
im: ru.InstanceMirror = ...

// Method mirror
scala> val m = ru.typeOf[class1].decl(ru.TermName("m")).asMethod
val m = ru.typeOf[C].decl(ru.TermName("m")).asMethod

scala> val mm = im.reflectMethod(methodZ)
mm: ru.MethodMirror = method mirror for def m: scala.Int...

scala> mm()
res0: Any = 2

// Field mirror
scala> val f = ru.typeOf[C].decl(ru.TermName("x")).asTerm.accessed.asTerm
f: ru.TermSymbol = value f

scala> val ff = im.reflectField(fieldX)
ff: ru.FieldMirror = field mirror for private[this] val f: scala.Int...

scala> ff.get
res1: Any = 2

scala> ff.set(3)

// Class mirror
scala> val c = ru.typeOf[class2].typeSymbol.asClass
class2: ru.ClassSymbol = class class2

scala> val cm = m.reflectClass(classC)
cm: ru.ClassMirror = class mirror for class2 (bound to null)

scala> val ctorC = ru.typeOf[class2].decl(ru.termNames.CONSTRUCTOR).asMethod
ctorC: ru.MethodSymbol = constructor class2

scala> val ctorm = cm.reflectConstructor(ctorC)
ctorm: ru.MethodMirror = constructor mirror for def <init>(x: scala.Int): class2 (bound to null)

scala> ctorm(2)
res2: Any = class2(2)

// Module mirror
scala> val o = ru.typeOf[object1.type].termSymbol.asModule
o: ru.ModuleSymbol = object object1

scala> val oo = m.reflectModule(objectC)
oo: ru.ModuleMirror = module mirror for object1 (bound to null)

scala> val obj = mm.instance
obj: Any = ...
```

### Type
#### `TypeTag`
编译器在编译过程中将完整的类型信息保存到`TypeTag`中，并将其携带到运行期。

```scala
// create typeTag from method
scala> val tt = ru.typeTag[Int]
tt: ru.TypeTag[Int] = TypeTag[Int]

// create typeTag from implicit parameter
scala> def paramInfo[T](x: T)(implicit tag: ru.TypeTag[T]): Unit = println(s"${tag.tpe}")
paramInfo: [T](x: T)(implicit tag: ru.TypeTag[T])Unit

scala> paramInfo(List(1, 2, 3))
List[Int]

// create typeTag from context bound
scala> def paramInfo[T: ru.TypeTag](x: T): Unit = println(s"${ru.typeTag[T].tpe}")
paramInfo: [T](x: T)(implicit evidence$1: ru.TypeTag[T])Unit

scala> paramInfo(List(1, 2, 3))
List[Int]
```

#### `ClassTag`
`ClassTag`只保留了部分类型信息，封装了擦除后的类型信息。

```scala
scala> import scala.reflect._
import scala.reflect._

// create classTag from method
scala> val ct = classTag[Int]
ct: scala.reflect.ClassTag[Int] = Int

// create classTag from implicit parameter
// type erased
scala> def paramInfo[T](x: T)(implicit tag: ClassTag[T]): Unit = println(s"$tag")
paramInfo: [T](x: T)(implicit tag: scala.reflect.ClassTag[T])Unit

scala> paramInfo(List(1, 2, 3))
scala.collection.immutable.List

// create classTag from context bound
// type erased
scala> def paramInfo[T: ClassTag](x: T): Unit = println(s"${classTag[T]}")
paramInfo: [T](x: T)(implicit evidence$1: scala.reflect.ClassTag[T])Unit

scala> paramInfo(List(1, 2, 3))
scala.collection.immutable.List
```

#### `WeakTypeTag`
`WeakTypeTag`保留了抽象类型信息。

```scala
scala> abstract class AbstractClass[T] {
     |   def getInnerWeakType[T: ru.WeakTypeTag](obj: T) = ru.weakTypeTag[T].tpe
     |   val list: List[T]
     |   val result = getInnerWeakType(list)
     | }
defined class AbstractClass

scala> val sc = new AbstractClass[Int] { val list = List(1,2,3) }
sc: AbstractClass[Int] = $anon$1@515940af

scala> sc.result
res0: reflect.runtime.universe.Type = scala.List[T]
```

## REFERENCE
1. [scala doc](https://docs.scala-lang.org)
2. [Scala Reflection － Mirrors,ClassTag,TypeTag and WeakTypeTag](https://cloud.tencent.com/developer/article/1014081)
