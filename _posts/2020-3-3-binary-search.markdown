---
layout: post
title:  "二分查找算法分析"
date:   2020-3-3
categories: 算法和数据结构
keywords: DataStructure BinarySearch
mathjax: true
author: wzx
---

从**搜索区间，终止条件和搜索策略**三个方面分析二分查找算法，目标查找和上下边界的查找。





## 目标查找
查找一个目标数，返回其索引，若没有查找到，返回-1。若存在重复的目标数，则会随机返回一个目标数的索引。
```c++
int binary_search(const int arr[], int start, int end, int key) {
    int ret = -1;

    int mid = 0;
    while (start <= end) {
        // 防止溢出
        mid = start + (end - start) / 2;
        if (arr[mid] < key){
            start = mid + 1;
        }else if (arr[mid] > key){
            end = mid - 1;
        }else {
            ret = mid;  
            break;
        }
    }

    return ret;
}
```

- **搜索区间**：$[start,end]$ 闭区间
- **搜索策略**：二分查找每次查找都会使**搜索区间缩小一半**。因为**序列是有序的**，所以判断中间值与目标值的大小，就可以将搜索区间缩小为 $[start,middle-1]$ 或者 $[middle+1,end]$
- **终止条件**
    - 找到目标数`arr[mid] == key`
    - `start > end`，即**搜索区间 $[start,end]$ 为空**，出现这种情况说明序列中不存在目标数。而**当`start == end`时，仍存在不为空的搜索区间，所应该继续搜索**

## 下界查找
查找一个目标数，返回其索引，若没有查找到，返回-1。若存在重复的目标数，则会返回最小的索引。

下面的两种写法的区别为**搜索区间**的不同，一种为 $[start,end]$，另一种为 $[start,end)$

### 写法一
```c++
int lower_bound(const int arr[], int start, int end, int key) {
    int ret = -1;
    int term = end;

    int mid = 0;
    while (start <= end) {
        mid = start + (end - start) / 2;
        if (arr[mid] < key){
            start = mid + 1;
        }else{
            end = mid - 1;
        }
    }

    if (start <= term && arr[start] == key){
        ret = start;
    }
    return ret;
}
```

- **搜索区间**：$[start,end]$ 闭区间，`end`参数应传入`arr.size()-1`
- **搜索策略**
    - 当`arr[mid] < key`时，搜索区间缩小为 $[middle+1,end]$ ，此时下界仍在搜索区间内
    - 当`arr[mid] > key`时，搜索区间缩小为 $[start,middle-1]$ ，此时下界仍在搜索区间内
    - 当`arr[mid] == key`时，搜索区间缩小为 $[start,middle-1]$ ，**使搜索区间右侧向下界逼近**
        - 若`arr[mid]`不是下界，自然被排除在区间之外
        - 若`arr[mid]`是下界，**下界被排除在区间的右侧**，但之后的迭代由于`arr[mid] < key`，搜索区间左侧会逼近下界，直至搜索空间变空
- **终止条件**
    - `start > end`，即**搜索区间 $[start,end]$ 为空**。由搜索策略可知，搜索区间的左侧和右侧都会向下界逼近，所以终止条件为搜索区间为空
    - 由区间搜索策略知，如果存在，下界为`start`。在目标数大于序列中的所有数时，需要考虑溢出的情况

### 写法二
```c++
int lower_bound(const int arr[], int start, int end, int key) {
    int ret = -1;
    int term = end - 1;

    int mid = 0;
    while (start < end) {
        mid = start + (end - start) / 2;
        if (arr[mid] < key){
            start = mid + 1;
        }else{
            end = mid;
        }
    }

    if (start <= term && arr[start] == key){
        ret = start;
    }
    return ret;
}
```

- **搜索区间**：$[start,end)$ 左闭右开，`end`参数应传入`arr.size()`
- **搜索策略**
    - 当`arr[mid] < key`时，搜索区间缩小为 $[middle+1,end)$ ，此时下界仍在搜索区间内
    - 当`arr[mid] > key`时，搜索区间缩小为 $[start,middle)$ ，此时下界仍在搜索区间内
    - 当`arr[mid] == key`时，搜索区间缩小为 $[start,middle)$ ，**使搜索区间右侧向下界逼近**
        - 若`arr[mid]`不是下界，自然被排除在区间之外
        - 若`arr[mid]`是下界，**下界处于右侧开区间内**，但之后的迭代由于`arr[mid] < key`，搜索区间左侧会逼近下界，直至搜索空间变空
- **终止条件**
    - `start == end`，即**搜索区间 $[start,end)$ 为空**。由搜索策略可知，如果存在，`start`为下界

## 上界查找
查找一个目标数，返回其索引，若没有查找到，返回-1。若存在重复的目标数，则会返回最大的索引。

下面的两种写法的区别为**搜索区间**的不同，一种为 $[start,end]$，另一种为 $[start,end)$

### 写法一
```c++
int upper_bound(const int arr[], int start, int end, int key) {
    int ret = -1;

    int mid = 0;
    while (start <= end) {
        mid = start + (end - start) / 2;
        if (arr[mid] > key){
            end = mid - 1;
        }else{
            start = mid + 1;
        }
    }

    if (end >= 0 && arr[end] == key){
        ret = end;
    }
    return ret;
}
```

- **搜索区间**：$[start,end]$ 闭区间
- **搜索策略**
    - 当`arr[mid] < key`时，搜索区间缩小为 $[middle+1,end]$ ，此时上界仍在搜索区间内
    - 当`arr[mid] > key`时，搜索区间缩小为 $[start,middle-1]$ ，此时上界仍在搜索区间内
    - 当`arr[mid] == key`时，搜索区间缩小为 $[middle+1,end]$ ，**使搜索区间左侧向上界逼近**
        - 若`arr[mid]`不是上界，自然被排除在区间之外
        - 若`arr[mid]`是上界，**上界被排除在区间的左侧**，但之后的迭代，搜索区间右侧会逼近上界，直至搜索空间变空
- **终止条件**
    - `start > end`，即**搜索区间 $[start,end]$ 为空**。由搜索策略可知，搜索区间的左侧和右侧都会向下界逼近，所以终止条件为搜索区间为空
    - 由区间搜索策略知，如果存在，上界为`end`。在目标数小于序列中的所有数时，需要考虑溢出的情况

### 写法二
```c++
int upper_bound(const int arr[], int start, int end, int key) {
    int ret = -1;

    int mid = 0;
    while (start < end) {
        mid = start + (end - start) / 2;
        if (arr[mid] > key){
            end = mid;
        }else{
            start = mid + 1;
        }
    }

    if (end >= 1 && arr[end-1] == key){
        ret = end - 1;
    }
    return ret;
}
```

- **搜索区间**：$[start,end)$ 左闭右开，`end`参数应传入`arr.size()`
- **搜索策略**
    - 当`arr[mid] < key`时，搜索区间缩小为 $[middle+1,end)$ ，此时上界仍在搜索区间内
    - 当`arr[mid] > key`时，搜索区间缩小为 $[start,middle)$ ，此时上界仍在搜索区间内
    - 当`arr[mid] == key`时，搜索区间缩小为 $[middle+1,end)$ ，**使搜索区间左侧向上界逼近**
        - 若`arr[mid]`不是上界，自然被排除在区间之外
        - 若`arr[mid]`是上界，**上界排除在闭区间的左侧**，但之后的迭代由于`arr[mid] > key`，搜索区间右侧会逼近上界，直至搜索空间变空
- **终止条件**
    - `start == end`，即**搜索区间 $[start,end)$ 为空**。由搜索策略可知，如果存在，`end-1`为下界

## 总结
- 特别注意搜索空间的开闭
- 在查找目标数时，可以在找到目标数时提前终止，其余的终止条件都为搜索空间为空
- 下界查找时，找到目标数时，应该将搜索区间向下缩减
- 上界查找时，找到目标数时，应该将搜索区间向上缩减
- 上界查找时，搜索区间为开区间的情况下，结果需要减1
- 上下界查找时，需要判断返回值是否是目标数

LeetCode上的[34. Find First and Last Position of Element in Sorted Array](https://leetcode.com/problems/find-first-and-last-position-of-element-in-sorted-array/)为二分搜索的上下界查找的典型应用，完整答案可以查看[题解](https://github.com/wzx140/LeetCode/blob/master/src/main/c%2B%2B/34.%20Find%20First%20and%20Last%20Position%20of%20Element%20in%20Sorted%20Array.cpp)

## REFERENCE
[1] [二分查找详解](https://labuladong.gitbook.io/algo/di-ling-zhang-bi-du-xi-lie/er-fen-cha-zhao-xiang-jie)  
[2] [34. Find First and Last Position of Element in Sorted Array Solution](https://leetcode.com/problems/find-first-and-last-position-of-element-in-sorted-array/solution/)
