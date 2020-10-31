---
layout: post
title:  "大数据日知录学习(四)：常用数据结构"
date:   2020-9-5 16:00
categories: 大数据理论与架构
tags: BigData
mathjax: true
author: wzx
---

- 目录
{:toc}


简要介绍大数据常用的数据结构





## Bloom Filter
> 布隆过滤器

二进制向量结构(空间利用率和时间效率)，检测集合中是否存在某元素，**存在误判但不存在漏判**


### 原理
**长度为m的位数组**存储集合信息，使用**k个独立的哈希函数**将数据映射到位数组空间。
```bash
# 集合A初始化布隆过滤器
BloomFilter(set A, hash_functions, integer m)
    # 位数组
    filter [1...m] = 0
    foreach a_i in A:
        foreach h_j in hash_functions:
            filter[h_j(a_i)] = 1
        end foreach
    end foreach

    return filter
```

元素a，若对于相同的k个哈希函数，对应位都为1，则存在于集合中
```bash
MembershipTest(element, filter, hash_functions)
    foreach h_j in hash_functions:
        if filter[h_i(element)] != 1 then
            return false
    end foreach
    
    return true
```

### 误判率
对于**集合大小n, 哈希函数个数k, 位数组大小m**，误判率为 $p_{fp}\approx (1-e^{-kn/m})^k$ ，最优的哈希函数个数为 $k=\frac{m}{n} ln2$ ，故已知集合大小 $n$ 的情况下，并且在期望的误判率 $P$ 下，位数组的大小为 $m=-\frac{nlnp}{(ln2)^2}$ 

### 改进
计数BF，位数组的一个比特位拓展为多个比特位，就可以增加删除集合成员的功能

## SkipList

可替代平衡树的数据结构，依靠随机数保持数据的平衡分布。在最坏情况下效率要低于平衡树，在大多数情况下仍然非常搞笑。增删改查的时间复杂度都是O(logn)。

<img src="{{ site.url }}/assets/img/2020-9-5-3.png" style="zoom:50%;" />

在有序链表的基础上，以随机概率给部分结点增加指针，指向更远的后方结点。如图所示，结点的层数对应着指针的个数。

### 查找

```bash
search(list, searchKey)
	x = list.head
	# 从当前层数遍历到最底层
	for i in [list.level, 1]:
		# 每层找到第一个大于或等于key的前一个结点
		while x.next[i].key < searchKey:
			x = x.next[i]
	# 底层的后一个结点即是目标值
	x = x.next[1]
	if x.key == searchKey: return x.val
	else: return failure
```

### 插入&更新

以下是随机生成层数的伪代码

```bash
randomLevel()
	level = 1
	# random() 生成[0, 1)的随机数
	# 以p的概率依次增加层数
	while random() < p and level < maxLevel:
		level += 1
	return level
```

以下是插入的伪代码

```bash
insert(list, searchKey, newValue)
	update = array[maxLevel]
	x = list.head
	# 从当前层数遍历到最底层
	for i in [list.lever, 1]:
		# 每层找到第一个大于或等于key的前一个结点
		while x.next[i].key < searchKey:
			x = x.next[i]
		update[i] = x
	x = x.next[1]
	# 更新
	if x.key == searchKey: x.val = newValue
	else:
		level = randomLevel
		if level > list.level:
			# 新结点大于当前层数，则要添加新层head
			for i in [list.level + 1, level]:
				update[i] = list.head
			list.level = level
		# 由下往上插入结点 update[i] -> x -> update[i].next[i]
		x = node(level, searchKey, newValue)
		for i in [1, level]:
			x.next[i] = update[i].next[i]
			update[i].next[i] = x
```

下图所示插入17的过程

<img src="{{ site.url }}/assets/img/2020-9-5-4.png" style="zoom:50%;" />

### 删除

删除的过程与插入类似

```bash
delete(list, searchKey)
	update = array[maxLevel]
	x = list.head
	# 从当前层数遍历到最底层
	for i in [list.lever, 1]:
		# 每层找到第一个大于或等于key的前一个结点
		while x.next[i].key < searchKey:
			x = x.next[i]
		update[i] = x
	# 找到要删除的结点
	x = x.next[1]
	if x.key == searchKey: 
		for i in [1, list.level]:
			# 要删除的结点在当前层没有，在更高的层也不会有
			if update[i].next[i] != x: break
			update[i].next[i] = x.next[i]
		# 更新当前的层数
		while list.level > 1 and list.head.next[list.level] == null:
			list.level -= 1
```



## LSM
> Log-structured Merge-tree

**将大量随机写转化为批量的顺序写**，提升磁盘写入速度，牺牲了读性能，可以使用布隆过滤器进行优化。

以LevelDB为例

![](https://gitee.com/wangzxuan/images_bed/raw/master/images/20200603113306.png)

- 存储结构：内存中的MemTable和磁盘上的各级SSTable文件就形成了LSM树
  - 内存
      - MemTable：跳表结构。用于数据的快速插入
      - Immutable MemTable：只读的跳表结构。在MemTable占用内存达到一定阈值之后，便转化为Immutable MemTable

  - 硬盘
      - Current：当前manifest文件名

      - Manifest：SSTable的元信息，各层级SSTable文件的Level，文件名，最大最小key

        <img src="{{ site.url }}/assets/img/2020-9-5-5.png" style="zoom:50%;" />

      - Log：MemTable的插入日志。用于系统崩溃，内存中的MemTable数据丢失后的数据恢复

      - SSTable：Immutable MemTable导出到磁盘的结果。层级key有序文件结构。第0层有minor compaction生成会出现key重叠现象，但其余层的文件里不会存在key重叠现象

- compaction

  - minor: 当MemTable达到一定大小时，转化为Immutable MemTable并按顺序写入level0的新SSTable文件中。不处理删除操作

    ![]({{ site.url }}/assets/img/2020-9-5-6.png)

  - major: 当某个Level下的SSTable文件数目超过一定数量时，从这个Level中选择一个文件(Level0 需要选择所有key重叠的文件)和高一Level的文件合并。

    - 轮流选择Level层的文件
    - Level+1层选择和Level层文件在key range上有重叠的所有文件进行合并
    - 使用多路归并排序，并判断这个KV是否被删除，形成一系列新Level+1层的多个文件
    - 删除Level层的那个文件和旧Level+1层的所有文件

    <img src="{{ site.url }}/assets/img/2020-9-5-7.png" style="zoom:50%;" />

## Merkle Hash Tree
主要用于海量数据下快速定位少量变化的数据内容，在BitTorrent，Git，比特币等中得到了应用。

<img src="{{ site.url }}/assets/img/2020-9-5-2.png" style="zoom:50%;" />

如图所示，**叶子结点是每个数据项的哈希值，分支结点则保存其所有子结点的哈希值，依次由下往上推，根结点保存整棵树的哈希值**。当某个数据项发生变化时，其对应的叶子结点的哈希值也会发生变化，其祖先结点也会跟着变化，这样在O(logn)时间内就能快速定位变化的数据内容。

## REFERENCE

1. 大数据日知录