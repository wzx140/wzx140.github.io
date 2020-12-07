---
layout: post
title:  "Flink: 数据流"
date:   2020-12-7 8:00
categories: Flink
keywords: Flink Dataset DataStream
mathjax: true
author: wzx
---

Flink中的有界流对应着`Dataset`为批处理和无界流对应着`DataStream`为流处理



## 数据流

### 数据流图

![]({{ site.url }}/assets/img/2020-12-7-1.png)

数据流图是一个由**有[状态]({% post_url 2020-12-7-flink-state %})的算子和可被其他算子消费的由算子产生的数据流**组成的。如图所示，数据流图以并行化的方式执行，所以**算子会根据并行度被并行化为多个子任务**，而数据流被拆分为一个或多个**流分区**，每个子任务对应于一个分区。

指定算子并行度的优先级从大到小为

- 算子级别: `DataSet.setParallelism();`

- 执行环境级别: 对当前任务的所有算子生效。`ExecutionEnvironment.setParallelism();`

- 提交任务级别: `flink run -p 10 WordCount.jar`

- 配置文件级别: `parallelism.default: 1`

### 数据交换

![]({{ site.url }}/assets/img/2020-12-7-2.png)

如图所示，OP1算子接收SRC1算子的输出，并将运算结果传递到下一层的SNK1算子。这三个算子由两个数据流进行连接，分别是IS1和IS3。

IS1是一种暂时性的中间结果，称为**流水线中间流**，生产者算子处理一个record之后，可以直接将其传递到消费者算子继续处理，SRC1算子和OP1算子可以并行运行。**流水线中间流通过阻塞队列来补偿短期的吞吐量波动，下层消费者会向上层生产者传播反压以维持流水线机制**。Flink将算子链用于**连续流以及批处理**程序中，以尽可能避免中间流的持久化。

IS3是需要将流序列化到非易失性存储的数据流，称为**阻塞数据流**，OP1算子的输出数据序列化到磁盘中之后，SNK1算子才能启动消费，这样**将生产与消费划分为了不同的执行阶段**。阻塞数据流要求生产者必须生产一定量的数据之后，才能用于下层的消费，它会先将积累的records存储到内存中，如果内存不够，那就序列化到磁盘中。

### 延迟与吞吐量

![]({{ site.url }}/assets/img/2020-12-7-3.png)

当在生产者端的一个record准备完成时，record会被序列化放入缓冲中，**算子间通过交换缓冲的方式来交换数据**，**当缓冲已满或达到超时条件时，生产者将缓冲发送给消费者**。如图所示，**缓冲容量大，吞吐量就高；缓冲超时短，延迟就低**。

### 控制事件

算子产生的控制事件随着其他record在数据流分区中传输。

- [checkpoint barriers]({% post_url 2020-12-7-flink-recover %}#容错)：用于**容错**。在流中插入checkpoint事件，会促使流将当前的状态保存下来，当发生故障后，可以直接使用上一次的checkpoint来恢复
- [WaterMark]({% post_url 2020-12-7-flink-time %}#watermark)：标识流分区中的事件时间
- iteration barriers：用于类似机器学习的迭代计算

### 迭代数据流

![]({{ site.url }}/assets/img/2020-12-7-4.png)

增量处理和迭代对于图计算和机器学习非常重要。一般的并行处理平台通过提交新job，增加新结点或者feedback边来支持迭代处理。Flink将迭代的头部和尾部算子用feedback边隐式连接。

## 基于数据流的流分析

Flink的`DataStream` API实现了无解流分析的框架，包含乱序事件处理，定义窗口以及维护和更新用户定义状态等

- [State(状态)]({% post_url 2020-12-7-flink-state %}): 在进行流式计算过程中的信息。流式计算在本质上是增量计算，也就是说需要不断地查询过去的状态。一般用作容错恢复和持久化
- [Time(时间)]({% post_url 2020-12-7-flink-time %}#time): Flink 支持了 Event time、Ingestion time、Processing time 等多种时间语义
- [Window(窗口)]({% post_url 2020-12-7-flink-time %}#window)
- [WaterMark(水印)]({% post_url 2020-12-7-flink-time %}#watermark)

## 基于数据流的批处理分析

有界数据集的批处理是无界数据流的流处理的特例。

- 批处理的容错不再使用检查点，而是完全重新执行流，因为输入是有界的。这使恢复的成本更多了，但由于避免了检查点，因此使常规处理的成本降低了
- `DataSet` API中的状态操作使用简化的内存数据结构，而不是键值对索引
- `DataSet` API引入特殊的算子
- `DataSet`可以优化调度阶段

## REFERENCE

1. CARBONE P, KATSIFODIMOS A, EWEN S, 等. Apache flink: Stream and batch processing in a single engine[J]. Bulletin of the IEEE Computer Society Technical Committee on Data Engineering, IEEE Computer Society, 2015, 36(4). 
2. flink官方文档](https://ci.apache.org/projects/flink/flink-docs-release-1.10/)