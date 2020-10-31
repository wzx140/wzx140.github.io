---
layout: post
title:  "Median of Two Sorted Arrays"
date:   2019-7-31
categories: 算法和数据结构
tags: LeetCode C++
mathjax: true
author: wzx
---

- 目录
{:toc}

LeetCode 上的[一个题目](https://leetcode.com/problems/median-of-two-sorted-arrays)。想到解法很容易，只是不满足题目要求的时间复杂度，难度为 *Hard*





## 题目
There are two sorted arrays **nums1** and **nums2** of size m and n respectively.  
Find the median of the two sorted arrays. The overall run time complexity should be O(log (m+n)).  
You may assume **nums1** and **nums2** cannot be both empty.

有两个排序数组分别为，长度为 m 的 **nums1** 和 长度为 n **nums2**。  
寻找两个排序数组的中位数。总的运行时间复杂应为 O(log (m+n))。  
你可以假定 **nums1** 和 **nums2** 不同时为空。

Example1:
```
nums1 = [1, 3]
nums2 = [2]

The median is 2.0
```

Example2:
```
nums1 = [1, 2]
nums2 = [3, 4]

The median is (2 + 3)/2 = 2.5
```

## 解法一
最开始想到的解法，找出数组中间的两个值。新建两个变量 `middle1`，`middle2`，遍历 **nums1** 和 **nums2**，每次将最小的数，如果索引是奇数就赋值给 `middle1`，偶数就赋值 `middle2`。最后，如果数组总长度是奇数则返回 `min(middle1, middle2)`，如果是偶数就返回 $\frac{middle1 + middle2}{2}$

这里有一点需要注意，总长度为奇数时，最后通过比较`min(middle1, middle2)`，得出中间值。这么做有两个原因，一是考虑数组总长度为 1 的情况，二是`middle1`和`middle2`没有特定的大小之分，必须判断一下这两个值的大小。

时间复杂度：`O(m+n)` ；空间复杂度：`O(1)`

```c++
double findMedianSortedArrays(vector<int>& nums1, vector<int>& nums2){
	int length1 = nums1.size();
	int length2 = nums2.size();
	int length = length1 + length2;

	// save the middle two elements
	int middle1 = 0;
	int middle2 = 0;

	int i = 0;
	int j = 0;
	int count = 0;
	while (count <= length / 2){
		// prevent overflow
		int num1 = std::numeric_limits<int>::max();
		int num2 = std::numeric_limits<int>::max();
		if(i < length1){
			num1 = nums1[i];
		}
		if(j < length2){
			num2 = nums2[j];
		}

		if(num1 < num2){
			i++;
			if((count + 1) % 2 == 0){
				middle2 = num1;
			}else{
				middle1 = num1;
			}
		}else{
			j++;
			if((count + 1) % 2 == 0){
				middle2 = num2;
			}else{
				middle1 = num2;
			}
		}

		count++;
	}

	if(length % 2 == 0){
		return (middle1 + middle2) / 2.0;
	}else{
		return std::max(middle1, middle2);
	}

}
```

## 解法二
题目要求，时间复杂度为 `O(log(m+n))`，再加上是排序数组，我们立即想到可以使用**二分查找**

设长度为m的数组A `A[0], A[1], ..., A[i-1], A[i], A[i+1], ..., A[m-1]`  
长度为n的数组B `B[0], B[1], ..., B[j-1], B[j], B[j+1], ..., B[n-1]`

将两个数组分为两个部分

```
      left_part          |        right_part
A[0], A[1], ..., A[i-1]  |  A[i], A[i+1], ..., A[m-1]
B[0], B[1], ..., B[j-1]  |  B[j], B[j+1], ..., B[n-1]
```

满足以下条件  
1. $B[j-1]\le A[i]$
2. $A[i-1]\le B[j]$
3. $i+j = \frac{m+n}{2}$，实际表达时为 `i+j = (m+n+1)/2`，向上取整，即`m+n`为奇数时，`right_part`会比`left_part`少一个数

于是，中位数就可以确定了。如果 `m+n` 是奇数时，中位数为 $max(A[i-1],B[j-1])$ ，如果 `m+n` 是偶数时，中位数为 $\frac{max(A[i-1],B[j-1]+min(A[i],B[j]))}{2}$

于是，问题便变成了，在 `[0，m]` 中寻找 `i`，满足 $B[j-1] \le A[i]$ 和 $A[i-1]\le B[j]$ ，其中 $j=\frac{m+n+1}{2}-i$ 并且 $m<n$ 保证 $j$ 非负

为什么这一题可以用二分法呢？我们都知道使用二分法的重要条件是**排序**，从本质上来说，这就要求目标 `i`，必须随着索引的增加和减少从两个方向偏离目标。在本题中，若目标 `i` 的索引变小，那么 $B[j-1] \le A[i]$ 这个限制条件将会原来越不满足；若目标 `i` 的索引变大，那么 另一个限制条件 $A[i-1]\le B[j]$ 将会原来越不满足，这就满足了从两个方向偏离目标，所以二分法对于本题是有效的。

```c++
double findMedianSortedArrays(vector<int>& nums1, vector<int>& nums2) {
	if (nums1.size() > nums2.size()) {
		std::swap(nums1, nums2);
	}

	int length1 = nums1.size();
	int length2 = nums2.size();

	//  The length of the smaller length array is zero
	if (length1 == 0) {
		return length2 % 2 ? nums2[length2 / 2] : (nums2[length2 / 2] + nums2[length2 / 2 - 1]) / 2.0;
	}

	int i = 0;
	int j = 0;

	int left = 0;
	// i search in 0~length1 not in 0~length1-1
	int right = length1;
	int delta = (length1 + length2 + 1) / 2;
	while (left <= right) {
		i = left + (right - left) / 2;
		j = delta - i;
		if (i < length1 && nums1[i] < nums2[j - 1]) {
			left = i + 1;
		} else if (i > 0 && nums1[i - 1] > nums2[j]) {
			right = i - 1;
		} else {
			break;
		}
	}

	// special boundary condition
	int maxLeft = 0;
	if (i == 0) {
		maxLeft = nums2[j - 1];
	} else if (j == 0) {
		maxLeft = nums1[i - 1];
	} else {
		maxLeft = std::max(nums1[i - 1], nums2[j - 1]);
	}

	int minRight = 0;
	if (i == length1) {
		minRight = nums2[j];
	} else if (j == length2) {
		minRight = nums1[i];
	} else {
		minRight = std::min(nums1[i], nums2[j]);
	}

	if ((length1 + length2) % 2 == 1) {
		return maxLeft;
	} else {
		return (maxLeft + minRight) / 2.0;
	}
}
```

有两点注意
1. `i` 的搜索范围为 `0~length`，而不是`0~length1-1`，考虑到 `nums1` 全在左边的情况。
2. 注意程序结尾的判断

## REFERENCE
[1][Median of Two Sorted Arrays](https://leetcode.com/problems/median-of-two-sorted-arrays/solution/) - LeetCode[EB/OL].
