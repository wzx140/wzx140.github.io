---
layout: post
title:  "Minimum Size Subarray Sum"
date:   2019-7-8
categories: 算法和数据结构
tags: LeetCode Java
mathjax: true
author: wzx
---

- 目录
{:toc}

LeetCode 上的[一个题目](https://leetcode.com/problems/minimum-size-subarray-sum/)。解决这个问题很容易，但很多方法不容易想到，而且自己最先想到的解法的时间复杂度比较大




## 题目
Given an array of n positive integers and a positive integer s, find the minimal length of a contiguous subarray of which the sum ≥ s. If there isn't one, return 0 instead.

给定长度为正整数n的数组和正整数s，找到连续子数列的最小长度，并且这个子列的总和大于等于s。 如果没有，则返回0。


Example:
```
Input: s = 7, nums = [2,3,1,2,4,3]
Output: 2
```
Explanation: the subarray [4,3] has the minimal length under the problem constraint.

解释：子列[4,3]拥有在问题限制下的最小长度

## 解法一
时间复杂度为 $O(n^2)$ ,比较容易想到，就是遍历所有子列，寻找符合条件的长度，这应该是最基本的解法
```java
public int minSubArrayLen(int s, int[] nums) {
    int length = nums.length + 1;
    for (int i = 0; i < nums.length - 1; i++) {
        int sum = 0;
        for (int j = i; j < nums.length; j++) {
            sum += nums[j];
            if (sum >= s) {
                length = Math.min(length, j - i + 1);
                break;
            }
        }
    }
    return length == nums.length + 1 ? 0 : length;
}
```

## 解法二
这里开始就需要动点脑筋了。这里首先构造了`sum`数组用来依次储存子列的和，即 `sum[i]=nums[0]+...+nums[i]`。

关键在代码里第二个循环的内部。原本我们的想法就是通过遍历找到一个**从下标 j 到 i 的子列的和大于等于 s**，即找到满足`sum[j] - sum[i] + nums[i-1] >= s`的**下标 j**，但是这样的话时间复杂度就与解法一相同

可以仔细想想，在有序数组中寻找一个特定值的话应该使用**二分查找**，这就是我们构造`sum`数组的原因，由`sum[j] - sum[i] + nums[i-1] >= s`，可以推导出`sum[j] >= s + sum[i] - nums[i-1]`，这样一来只需在数组中找到值比`s + sum[i] - nums[i-1]`大一点或者相等的元素即可，这样的做法与普通的二分查找不同，只要一些微调即可。二分查找时间复杂度为 $O(logn)$ ，所以总时间复杂度为 $O(nlogn)$ ，比解法一要优越一点

```java
public int minSubArrayLen(int s, int[] nums) {

    int[] sum = new int[nums.length + 1];
    sum[0] = 0;
    for (int i = 1; i < nums.length + 1; i++) {
        sum[i] = sum[i - 1] + nums[i - 1];
    }

    int length = nums.length + 1;
    for (int i = 1; i < sum.length; i++) {
        int index = binary_search(sum, i, sum.length - 1, s + sum[i] - nums[i -1]);
        if (index != -1) {
            length = Math.min(length, index - i + 1);
        }
    }
    return length == nums.length + 1 ? 0 : length;
}

private int binary_search(int[] array, int start, int end, int key) {
    while (start <= end) {
        int middle = (end - start) / 2 + start;
        if (array[middle] > key) {
            end = middle - 1;
        } else if (array[middle] < key) {
            start = middle + 1;
        } else {
            return middle;
        }
    }
    return start > array.length - 1 ? -1 : start;
}
```

## 解法三
使用**两个指针**，又称**虫取法**，顾名思义，两个指针标记着虫子的头和尾，在原数组上移动。
快指针（头部）先移动，找到总和大于等于s的子列，再尽可能多的移动慢指针（尾部）但总和必须大于等于s。这样首尾依次移动直到头部到达底端，尾部到达总和大于等于s的临界状态。

简单分析一下便知，最坏的情况，每个元素被快指针遍历一次，被慢指针遍历一次，所以时间复杂度为 $O(n)$

```java
public int minSubArrayLen3(int s, int[] nums) {
    int left = 0;
    int length = nums.length + 1;
    int sum = 0;
    for (int i = 0; i < nums.length; i++) {
        sum += nums[i];
        while (sum >= s) {
            length = Math.min(length, i - left + 1);
            sum -= nums[left++];
        }
    }
    return length == nums.length + 1 ? 0 : length;
}
```
