---
layout: post
title: "线程安全的单例模式"
date:   2021-1-10 8:00
categories: Java
keywords: 单例, 并发
mathjax: false
author: wzx
---

许多时候整个系统只需要拥有一个的全局对象，这样有利于协调系统整体的行为。




## 饿汉式

在类初始化时，单例对象已经自行实例化。**所以全局的单例实例在类装载时构建**
```java
public class Singleton {

  private static Singleton instance = new Singleton();

  // 私有构造函数，外部不可调用
  private Singleton() {}

  public static Singleton getInstance() {
    return instance;
  }

}
```

## 懒汉式

在第一次调用时，单例对象进行实例化。所以**全局的单例实例在第一次被使用时构建**。**字段延迟初始化降低了初始化类或创建实例的开销，但增加了访问被延迟初始化的字段的开销**。除非确有必要，在大多数时候，正常的初始化要优于延迟初始化。

### 单线程

这种方法是线程不安全的

```java
public class Singleton {

  private static Singleton instance = null;

  // 私有构造函数，外部不可调用
  private Singleton() {}

  public static Singleton getInstance() {
    if (instance == null) {
      instance = new Singleton();
    }
    return instance;
  }
}
```

### 多线程

通过**双重检查锁定**确保，多个线程试图在同一时间创建对象时，只有一个线程能创建对象，在对象创建好之后，不需要获取锁，直接返回已创建好的对象。

```java
public class Singleton {

  private static Singleton instance = null;

  // 私有构造函数，外部不可调用
  private Singleton() {}

  public static Singleton getInstance() {
    if (instance == null) {
      synchronized (Singleton.class) {
        if (instance == null) {
          instance = new Singleton();
        }
      }
    }
    return instance;
  }

}
```

以上代码存在一个问题，由于**重排序**可能会导致，代码读取到`instance`，判断不为`null`时，**`instance`引用的对象有可能还没有完成初始化**。

对于`instance = new Singleton()`来说可以分解为以下三个伪代码

1. `memory = allocate(); ` 分配对象的内存空间 
2. `ctorInstarce(memory);` 初始化对象 
3. `instance = memory;` 设置`instance`指向刚分配的内存地址

步骤2和步骤3可能发生重排序，即**`instance`指向刚分配的内存地址时，对象初始化没有完成**。这时，如果其他线程判断了`instance != null`而没有获取锁直接将`instance`返回，那么返回的将是一个未初始化完成的对象。

#### 双重检查锁定

使用`volatile`修饰`instance`可以**阻止`instance = new Singleton()`语句重排序**

```java
public class Singleton {

  private volatile static Singleton instance = null;

  // 私有构造函数，外部不可调用
  private Singleton() {
  }

  public static Singleton getInstance() {
    if (instance == null) {
      synchronized (Singleton.class) {
        if (instance == null) {
          instance = new Singleton();
        }
      }
    }
    return instance;
  }

}
```

#### 类初始化

在执行类的初始化期间，JVM会去获取一个锁。这个锁可以**同步多个线程对同一个类的初始化**

```java
public class Singleton {

  // 私有构造函数，外部不可调用
  private Singleton() {}

  public static Singleton getInstance(){
    return singletonHolder.instance;
  }
  
  private static class singletonHolder{
    private static Singleton instance = new Singleton();
  }
  
}
```

如图所示，即使`instance = new Singleton()`发生重排序，也不会出现返回未完成初始化对象的情况

![]({{ site.url }}/assets/img/2021-1-10-1.png)

## REFERENCE

1. Java并发编程的艺术