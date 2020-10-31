---
layout: post
title:  "深入理解Java String"
date:   2020-3-12
categories: Java
tags: Java String
mathjax: false
author: wzx
---

- 目录
{:toc}

解释Java String的不可变性以及字符串常量池。




## 不可变性
- `String`的数据保存在内部**不可变的字符数组**之中，`private final char value[]`
	- 私有访问权限保证了只有`String`内部方法才能访问数据
	- `final`保证了`value`引用的地址不可变
- `String`类被`final`修饰，**不可继承**，避免内部数据被其他类继承从而遭到破坏
- `String`类的所有方法都不会改变`value`的值

这些特点导致了`String`封装的字符串数据是不可改变的。

## 字符串比较
### 全局字符串池
字符串的分配，耗费高昂的时间与空间代价，作为最基础的数据类型，大量频繁的创建字符串，极大程度地影响程序的性能，JVM为了提高性能和减少内存开销，开辟了一个特殊的空间，全局字符串池，**JDK1.6字符串池在方法区中，JDK1.7在堆中**。

**字符串池中保留堆中字符串常量的对象引用**。在HotSpot VM里实现字符串池功能的是一个 **`StringTable`类，它是一个哈希表**，里面存的是驻留字符串(字面量)的**引用**。这个`StringTable`在每个HotSpot VM的实例只有一份，被所有的类共享。**JDK1.6字符串池中直接保存字符串实例，而JDK1.7保存对象引用**。

### 字符串创建
- 使用new关键字创建字符串，`String s1 = new String("abc");`
  - 由于指定了字面量"abc"，首先在字符串池中寻找"abc"的引用，若有则返回引用，否则在堆中创建对象"abc"(1)，并在字符串中保留对象引用
  - 使用了new关键字，在堆中创建字符串对象"abc"(2)
  - 此处"abc"(1)与"abc"(2)都是堆中的字符串对象，但是不是同一个
- 使用字面量创建字符串，`String s2 = "abc";`
	- 首先在字符串池中寻找"abc"的引用，若有则返回引用，否则在堆中创建对象"abc"，并在字符串中保留对象引用
	- `s2`保存"abc"的对象引用，不会在堆上创建新的对象
- `s.intern()`
	- 在字符串池中寻找字符串对象的引用，若有则返回引用，否则在字符串池中保存`s`的对象引用

#### 字符串串联
- 如果含有String对象，则**底层是使用StringBuilder实现的拼接的**，所以会**在堆中创建一个新实例**
```java
String str1 ="str1";
String str2 ="str2";
String str3 = str1 + str2;
```
- 如果只有字面量或者常量或基础类型变量，则会**直接编译为拼接后的字符串**，**字面量参数不会保存在常量池中**。
```java
// 等同于String str1 = "aabb"
String str1 = "aa" + "bb";
```

### 字面量何时进入常量池
- 对于HotSpot VM，**加载类的时候，字符串字面量会进入到当前类的runtime常量池，不会进入全局的字符串常量池**。
- 在字面量赋值的时候，会翻译成字节码ldc指令，ldc指令触发lazy resolution动作到当前类的runtime常量池去查找该index对应的项
- 如果该项尚未resolve则resolve之，并返回resolve后的内容。
- 在遇到String类型常量时，resolve的过程如果发现StringTable已经有了内容匹配的java.lang.String的引用，则直接返回这个引用;
- 如果StringTable里尚未有内容匹配的String实例的引用，则会在Java堆里创建一个对应内容的String对象，然后在StringTable记录下这个引用，并返回这个引用出去。

### 实例
```java
String s = new String("2");
s.intern();
String s2 = "2";
// JDK1.6:false
// JDK1.7:false
System.out.println(s == s2);

String s3 = new String("3") + new String("3");
s3.intern();
String s4 = "33";
// JDK1.6:false
// JDK1.7:true
System.out.println(s3 == s4);
```
#### JDK1.6
![]({{ site.url }}/assets/img/2020-3-12-1.png)
- `String s = new String("2");`**创建了两个对象**，一个在堆中的`String`对象(s指向)，一个是在常量池中的"2"对象。
- `s.intern();`在常量池中寻找与s变量内容相同的对象，发现已经存在内容相同对象"2"，返回对象"2"的地址。
- `String s2 = "2";`使用字面量创建，在常量池寻找是否有相同内容的对象，发现有，**返回对象"2"的地址**。
- `s`与`s2`，一个是堆中的`String`对象，一个是常量池中的"2"对象，不一样。

---

- `String s3 = new String("3") + new String("3");`**创建了两个对象**，一个在堆中的`String`对象(s3指向)，一个是在常量池中的"3"对象。不讨论匿名对象。
- `s3.intern();`在常量池中寻找与s3变量内容相同的对象，没有发现"33"对象，**在常量池中创建"33"对象，返回"33"对象的地址**。
- `String s4 = "33";`使用字面量创建，在常量池寻找是否有相同内容的对象，发现有，**返回对象"33"的地址**。
- `s3`与`s4`，一个是堆中的`String`对象，一个是常量池中的"33"对象，不一样。

#### JDK1.7
![]({{ site.url }}/assets/img/2020-3-12-2.png)
- `String s = new String("2");`**创建了两个对象**，一个在堆中的`String`对象(s指向)，一个是在堆中的"2"对象，并在常量池中保存"2"对象的引用地址。
- `s.intern();`在常量池中寻找与s变量内容相同的对象，发现已经存在内容相同对象"2"，返回对象"2"的地址。
- `String s2 = "2";`使用字面量创建，在常量池寻找是否有相同内容的对象，发现有，**返回对象"2"的地址**。
- `s`与`s2`，一个是堆中的`String`对象，一个是堆中"2"对象(常量池中存在其对象引用)，不一样。

---

- `String s3 = new String("3") + new String("3");`**创建了两个对象**，一个在堆中的`String`对象(s3指向)，一个是在堆中的"3"对象，并在常量池中保存"3"对象的引用地址。不讨论匿名对象。
- `s3.intern();`在常量池中寻找与s3变量内容相同的对象，没有发现"33"对象，**将s3指向的`String`对象的地址保存到常量池中，返回`String`对象的地址**。
- `String s4 = "33";`使用字面量创建，在常量池寻找是否有相同内容的对象，发现有，**返回常量池中`String`对象的地址**。
- `s3`与`s4`，都指向堆中的`String`对象。

## REFERENCE
[1] [Java中几种常量池的区分](http://tangxman.github.io/2015/07/27/the-difference-of-java-string-pool/)  
[2] [java基础：String — 字符串常量池与intern(二）](https://juejin.im/post/5c160420518825235a05301e)