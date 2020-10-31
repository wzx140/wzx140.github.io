---
layout: post
title:  "Java中的位运算"
date:   2019-10-15
categories: Java
tags: Java bitOperation
mathjax: true
author: wzx
---

- 目录
{:toc}

在看 **双端队列`ArrayDeque`** 的源码时，发现了一些很奇怪的位运算操作，需要绕些弯子才能理解





## 位运算符
简单说明一下 `Java` 的位运算符
- `<<`：按**位**左移，低位补0
- `>>`：按**位**右移，高位补0，负数补1
- `<<<`：按**位**左移，低位补0，**不算符号位**
- `>>>`：按**位**右移，高位补0，**不算符号位**
- `&`：按位与，按**位**做**与**运算，`1&0 = 0&1 = 0&0 =0, 1&1 = 1`
- `|`：按位或，按**位**做**或**运算，`1|0 = 0|1 = 1|1 =1, 0|0 = 0`
- `^`：按位异或，按**位**做**异或**运算，相同为0，不相同为1
- `~`：按位非，按**位**做**非**运算

## 源码分析
`ArrayDeque` 内部使用顺序表保存元素，并有两个数据域 `head` 和 `tail` 分别指向头和尾。由于没有保存元素个数的数据域，所以为保证能正确判断顺序表满与空的状态，**head指向当前头结点，tail指向当前尾结点的后一个元素**

### 移位
```java
private void doubleCapacity() {
    assert head == tail;
    int p = head;
    int n = elements.length;
    int r = n - p; // number of elements to the right of p
    int newCapacity = n << 1;
    if (newCapacity < 0)
        throw new IllegalStateException("Sorry, deque too big");
    Object[] a = new Object[newCapacity];
    System.arraycopy(elements, p, a, 0, r);
    System.arraycopy(elements, 0, a, r, p);
    elements = a;
    head = 0;
    tail = n;
}
```
这是当存储空间不够时，扩展更大容量的顺序表的方法，`elements.length` 是通过**左移**位运算符得到的，所以`elements.length` 总是 `00...100...00`这种模式，从而 `elements.length-1` 总是 `00...0011...11`，非零位集中在最后

**`elements.length-1`的这种表现形式是之后**按位与**运算符起作用的前提条件**。即使使用`ArrayDeque(int numElements)`去初始化双端队列，该方法内部仍然会调用 `calculateSize(int numElements)` 使得`elements.length`总是 $2^n$

### 按位与

```java
public int size() {
    return (tail - head) & (elements.length - 1);
}
```
如果 `tail>head`，那么返回的就是 `tail-head`，就是队列的长度

如果  `tail<head`。那么 `tail-head` 的值就是**剩余空间的负数**。负数用[补码](https://baike.baidu.com/item/%E8%A1%A5%E7%A0%81)表示，计算法则为对应的正数取反加1。

我们可以这样想，如果补码只是取反，那么由于`elements.length-1`的特殊性，`(tail - head) & (elements.length - 1)`就相当于`(elements.length - 1) - |tail - head|`。由于`elements.length-1`，最终返回的长度会减小1，但是补码还需要进行+1运算，所以这个减小的值又补偿回来了

似乎有点绕，举个例子
```
tail-head = -3D = 11...1101B (+3D=00...0011B)
length-1  = 15D = 00...1111B
结果      = 13D = 00...1101B

实际长度   = length - |tail - head| = 16 - |-3| = 13
```

再看另外一处

```java
public void addFirst(E e) {
    if (e == null)
        throw new NullPointerException();
    elements[head = (head - 1) & (elements.length - 1)] = e;
    if (head == tail)
        doubleCapacity();
}
```
如果 `head>0`，做**按位与**运算时，`(head-1) & (elements.length-1)=head-1`始终成立，所以新加入的元素就添加进了顺序表的前面一位

如果 `head=0`，-1的补码为`11...11`，自然 `(head-1) & (elements.length-1)=elements.length-1`，所以当head处于顺序表的首位时，新加入的元素就会放入顺序表的末尾

在 `addLast()` `pollFirst()` 等其他方法中也使用了**按位与**运算，作用类似，可以自己试着举例子来理解
