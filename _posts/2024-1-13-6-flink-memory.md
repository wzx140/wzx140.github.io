---
layout: post
title:  "Flink 剖析(六) 内存管理与类型系统"
date:   2024-1-13
categories: Flink
keywords: Flink 
mathjax: true
author: wzx
---

Flink 中内存管理与类型系统



## 内存管理

### 内存模型

![]({{ site.url }}/assets/img/2024-1-13-6-3.png){:width="200"}

- 堆内内存
  - 框架内存：TM 本身所占用的堆内内存，不记入 Slot 的资源中
  - Task 内存：Task 执行用户代码时所使用的堆内内存

- 堆外内存
  - 直接内存
    - 框架内存：TM 本身所占用的堆外内存，不记入 Slot 的资源中
    - Task 内存：Task 执行用户代码时所使用的堆外内存
    - 网络缓冲内存：Task 间网络传输使用到的缓存
  - 托管内存：Flink 管理的堆外内存。在批处理中用在排序、Hash 表和中间结果的缓存中，在流计算中作为 RocksDBStateBackend 的内存

### 内存数据结构

#### MemorySegment

**MemorySegment 是 Flink 的内存抽象的最小分配单元**。默认情况下，一个MemorySegment 对应着一个 32 KB 大小的内存块。这块内存既可以是堆内内存(Java 的 byte 数组)，也可以是堆外内存(基于 Netty 的 DirectByteBuffer)

#### DataView

MemorySegment 之上存储数据访问视图，读取抽象为 DataInputView，数据写入抽象为 DataOutputView。有了这一层，上层使用者无须关心  MemorySegment 的细节，**该层会自动处理跨 MemorySegment 的读取和写入**

#### NetworkBuffer

NetworkBuffer 包装了 MemorySegment，用来**处理各个 TM 之间的数据传递**

#### BufferPool

BufferPool 用来管理 Buffer 的申请、释放、销毁、可用 Buffer 通知等。

每个 TM 只有一个 NetworkBufferPool，同一个 TM 上的 Task 共享NetworkBufferPool，在 TM 启动的时候，就会创建 NetworkBufferPool，为其分配内存

#### MemoryManager

MemoryManager 用来理堆外的托管内存。在批处理中用在排序、Hash表和中间结果的缓存 中，在流计算中作为RocksDBStateBackend的内存

## 类型系统

### 逻辑类型

- **物理类型系统面向开发者**，指的是 Java Class
- 逻辑类型系统是描述物理类型的类型系统，能够**对物理类型就行序列化/反序列化**。对于常见的物理类型，Flink 提供了内置的序列化方法，其他类型默认交给 kryo 进行序列化，也可以自定义。
  - 针对 DataStream/DataSet API 的 TypeInformation 类型系统
  - 针对 SQL 的 LogicalTypes 类型系统

![]({{ site.url }}/assets/img/2024-1-13-6-1.png)

### 类型推断

Flink 应用使用反射机制获取 Function 的输入和输出类型。对于一些带有泛型的类型， **Java 泛型的类型擦除机制会导致 Flink 在处理 Lambda 表达式的类型推断时不能保证一定能提取到类型**。

在开发某些带有泛型返回值的 transformation 时，需要传入 TypeHint 来获取泛型的类型信息。

## SQL Row

接口 BaseRow 定义了针对表中一行数据的处理方法 getter(index)/setter(index) 等，有以下实现类

- BinaryRow: 二进制行式存储，分为定长部分和不定长部分，定长部分只能在一个 MemorySegment 内，来提升读写字段的速度
- NestedRow: 与 BinaryRow 的内存存储结构一样，区别在于 NestedRow的定长部分可以跨 MemorySegment
- UpdatableRow: 该类型的Row比较特别，其保存了该行所有字 段的数据，更新字段数据的时候不修改原始数据，而是使用一个数组 记录被修改字段的最新值。读取数据的时候，首先判断数据是否被更 新过，如果更新过则读取最新值，如果没有则读取原始值
- ObjectArrayRow: 使用对象数组保存一整行的数据
  - GenericRow: 存储的数据类型是原始类型
  - BoxedWrapperRow: 存储的数据类型是可序列化和可比较大小的对象类型
- JoinedRow: 逻辑上合并的两个 BaseRow，物理上没有合并

重点介绍以下 BinaryRow 的存储结构

![]({{ site.url }}/assets/img/2024-1-13-6-2.png)

- 定长部分
  - 头信息区：1 byte
  - 空值索引(Null Bit Set)：8 bytes 对齐，标记 Null 的位置
  - 定长字段值：保存基本类型和 8 bytes 长度以内字段的值
  - 变长字段索引：保存变长类型和大于 8 bytes 长度字段的 offset 和 length
- 变长部分
  - 变长字段值：保存变长类型和大于 8 bytes 长度字段的值

## REFERENCE

1. [flink官方文档](https://ci.apache.org/projects/flink/flink-docs-release-1.10/)

