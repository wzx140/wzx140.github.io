---
layout: post
title:  "Pointer and const"
date:   2018-11-4
categories: Others
tags: C++ Pointer
mathjax: false
author: wzx
---

- 目录
{:toc}

将const用于指针总能产生一些神奇的效果





## pointer to const
### 特性

```c++
int age = 39;
const int *p = &age;
*p++;           // INVALID
age = 1;        // VALID

const int boot = 2;
int *pp;
p = &boot;      // VALID
pp = &boot      // INVALID
```
要注意以下几点：
1. *p* 既可以指向常量也可以指向非常量(包括指针)，但不可以通过 *p* 来改变其指向的值。如果指向的值不是常量，则可以通过本身来改变
2. 普通指针不能指向 *const变量*


> 第二点说明了，常量表明值不能修改，故如果让一个普通指针来指向常量，就不能体现原有的特点，对于开发者来说就容易写出bug

### 在函数中的使用
```c++
int sum(int arr[], int n);
...
const int a[2] = {1, 2};
sum(a, 2);      // INVALID
```
*int arr[]* 作为函数的形参来说就相当于 int *arr ，正如前文所说，普通指针自然不能指向常量
### 尽可能使用const
1. 避免修改不想改变的数据
2. 一些会改变参数的函数将不能接受 *const变量* ，避免出错的可能

## const pointer
从这里开始事情开始变得越来越有趣
```c++
int age = 39;
int aged = 40;
const int *p = &age;
p = &aged;      // VALID

int * const pp = &age; //a const pointer
pp = &aged;     // INVALID
```
这是可以使用简单一句话来描述这种现象  
*pp* 以及 \*p 都是 *const*，而 \*pp 和 *p* 不是  

  参考文献：C++ Primer Plus
