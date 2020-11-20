---
layout: post
title: "Spark源码阅读(三十四): SparkSQL之Tungsten内存优化应用"
categories: Spark
date:   2020-11-20
keywords: Spark, SparkSQL, Tungsten
mathjax: false
author: wzx
---

基于[Tungsten内存管理方式]({% post_url 2020-9-24-calculation-memory %})实现的基本数据类型和数据结构





## ByteArray

`ByteArray`作为final类型，提供了以下对byte数组操作的静态工具方法

- `writeToMemory()`: 调用`Platform.copyMemory()`方法将字节数组写入特定已分配的内存空间
- `getPrefix()`: 从byte数组计算得到一个64位的整型用于排序
- `subStringSQL()`: 获得子byte数组
- `concat()`: 连接多个byte数组

## LongArray

由[Tungsten内存管理方式]({% post_url 2020-9-24-calculation-memory %})实现的长整型数组，其构造参数只有`MemoryBlock`。支持数据存储在堆内和堆外，提供了get, set等方法。

## UTF8String

由[Tungsten内存管理方式]({% post_url 2020-9-24-calculation-memory %})实现的字符串类型，负责将UTF-8编码的字符串类型编码为`Array[Byte]`，用于字符串的搜索和字符串比较等场景，提供了各种常用的字符串处理功能，包括基本的`startsWith`、`endsWith`、`toUpperCase`等方法。

## BytesToBytesMap

由[Tungsten内存管理方式]({% post_url 2020-9-24-calculation-memory %})实现的字节数组哈希表，只能添加和更新元素。相比`HashMap`，能够有效降低 JVM对象存储占用的空间和GC开销。**被`UnsafeHashedRelation`使用，用于基于哈希的join。被`UnsafeFixedWidthAggregationMap`使用，用于基于哈希的聚合**

![]({{ site.url }}/assets/img/2020-11-20-1.png)

如图所示，**成员属性`longArray: LongArray`负责保存key信息，每个key会占用两个long空间分别存放页号和页内偏移量的编码值`fullKeyAddress`(可以由`TaskMemoryManager`解码得到)和key的哈希值**，哈希码由key的keyBase, keyOffset, keyLength生成。**成员属性`dataPages: LinkedList<MemoryBlock>`维护着申请到的页表**。 **每条记录包含占用空间信息，key和value值，最后一部分的指针指向下一个数据，以以应对哈希冲突的情况**。

如下代码所示，基本对应着上图所示的查找过程，其中`hash`由`Murmur3_x86_32.hashUnsafeWords(keyBase, keyOffset, keyLength, 42)`生成，`loc: Location`包装查找到的值并返回。**使用步长为1的线性探查去处理哈希冲突**。

```scala
public void safeLookup(Object keyBase, long keyOffset, int keyLength, Location loc, int hash) {
  assert(longArray != null);

  if (enablePerfMetrics) {
    numKeyLookups++;
  }
  int pos = hash & mask;
  int step = 1;
  while (true) {
    if (enablePerfMetrics) {
      numProbes++;
    }
    if (longArray.get(pos * 2) == 0) {
      // This is a new key.
      loc.with(pos, hash, false);
      return;
    } else {
      long stored = longArray.get(pos * 2 + 1);
      if ((int) (stored) == hash) {
        // Full hash code matches.  Let's compare the keys for equality.
        loc.with(pos, hash, true);
        if (loc.getKeyLength() == keyLength) {
          final boolean areEqual = ByteArrayMethods.arrayEquals(
            keyBase,
            keyOffset,
            loc.getKeyBase(),
            loc.getKeyOffset(),
            keyLength
          );
          if (areEqual) {
            return;
          }
        }
      }
    }
    pos = (pos + step) & mask;
    step++;
  }
}
```

数据插入和扩容的逻辑基本与[`AppendOnlyMap`]({% post_url 2020-9-28-calculation-dataStructure %}#appendonlymap)相同，只不过底层使用基于[Tungsten内存管理方式]({% post_url 2020-9-24-calculation-memory %})实现，这里不再分析。

## UnsafeFixedWidthAggregationMap

**用于在聚合值宽度固定的情况下执行聚合**。主要用法如下，调用`getAggregationBufferFromUnsafeRow()`根据key得到对应的聚合缓冲区，内部依赖了`map: BytesToBytesMap`

```scala
public UnsafeRow getAggregationBufferFromUnsafeRow(UnsafeRow key, int hash) {
  // Probe our map using the serialized key
  final BytesToBytesMap.Location loc = map.lookup(
    key.getBaseObject(),
    key.getBaseOffset(),
    key.getSizeInBytes(),
    hash);
  if (!loc.isDefined()) {
    // This is the first time that we've seen this grouping key, so we'll insert a copy of the
    // empty aggregation buffer into the map:
    boolean putSucceeded = loc.append(
      key.getBaseObject(),
      key.getBaseOffset(),
      key.getSizeInBytes(),
      emptyAggregationBuffer,
      Platform.BYTE_ARRAY_OFFSET,
      emptyAggregationBuffer.length
    );
    if (!putSucceeded) {
      return null;
    }
  }

  // Reset the pointer to point to the value that we just stored or looked up:
  currentAggregationBuffer.pointTo(
    loc.getValueBase(),
    loc.getValueOffset(),
    loc.getValueLength()
  );
  return currentAggregationBuffer;
}
```

## UnsafeExternalSorter

实现了缓存敏感计算。如图所示，常规的做法是每个键值对record中有一个指针指向该record，对两个record 比较时先根据指针定位到实际数据，然后对实际数据进行比较，**由于这个操作涉及的是内存的随机访问，缓存本地化(空间局部性)会变得很低**。针对该缺陷，**缓存友好的存储方式会将key和record指针放在一起，以key为前 缀，避免内存的随机访问**。

![]({{ site.url }}/assets/img/2020-11-20-2.png)

对于`UnsafeExternalSorter`来说排序功能主要依赖于`UnsafeInMemorySorter`，类似于[`ShuffieExternalSorter`和`ShuffleInMemorySorter`]({% post_url 2020-9-28-calculation-sorter %}#shuffleexternalsorter)的关系，**区别为数据的插入分为两部分， 一部分是调用`Platform`方法进行真实数据的插入，另一部分是其索引，真实数据插入的地址和其prefix值的插入，当记录只用prefix来比较大小时，真实数据的排序就转换为索引的排序**。**被`UnsafeExternalRowSorter`使用，实现了`InternalRow`的排序，用于`SortExec`物理计划的排序。被`UnsafeKVExternalSorter`使用，实现了键值对数据的排序功能，用于`HashAggregateExec`物理计划中保存溢出聚合缓冲区**。

`UnsafeExternalSorter`中使用`TaskMemoryManager`和其他属性跟踪当前的内存使用和分配情况，使用链表`allocatedPages`保存申请到的`MemoryBlock`，以及使用链表维护数据溢出对象`UnsafeSorterSpillWriter`。

如下图所示展示了`UnsafeExternalSorter`的构造参数和写入数据的接口。其中**`UnsafeInMemorySorter`存储数据指针和数据前缀**，对两条记录进行排序时，首先判断两条记录指针的prefix是否相等，否则根据数据指针得到的实际数据再进行进一步的比较。 相对于实际数据，pointer和prefix占用的空间都比较小，**在比较时遍历较小的数据结构更有利于提高cache命中率(时间局部性)**。抽象类`RecordComparator`用来比较两条记录的大小。抽象类`PrefixComparator`提供了前缀比较功能，其抽象子类`RadixSortSupport`定义了基数排序的接口。

![]({{ site.url }}/assets/img/2020-11-20-3.png)

## REFERENCE

1. Spark SQL内核剖析