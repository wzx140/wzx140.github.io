---
layout: post
title:  "Flink 剖析(五) checkpoint"
date:   2024-1-13
categories: Flink
keywords: Flink 2PC Barrier Consistence
mathjax: true
author: wzx
---

Flink中精准一次的容错机制 10.8/10.9






## 一致性

### 一致性语义

- **最多一次(At-most-Once)**: 用户的数据只会被处理一次，不管成功还是失败，不会重试也不会重发。
- **至少一次(At-least-Once)**: 这种语义下，系统会保证数据或事件至少被处理一次。如果中间发生错误或者丢失，那么会从源头重新发送一条然后进入处理系统，所以同一个事件或者消息会被处理多次。
- **精确一次(Exactly-Once)**: 每一条数据只会被精确地处理一次，不会多也不会少。

### 2PC

通过2PC保证source和sink的精确一次(Flink作业失败)，需要实现以下四个方法

- `beginTransaction()`: 开始一个事务，返回事务信息的句柄。在开启事务之前，在目标文件系统的临时目录中创建一个临时文件，后面在处理数据时将数据写入此文件
- `preCommit()`: 在预提交阶段，刷写然后关闭文件，并且为属于下一个检查点的任何后续写入启动新事务
- `commit()`: 在提交阶段，将预提交的文件原子性移动到真正的目标目录中，这会增加输出数据可见性的延迟
- `abort()`: 在中止阶段，删除临时文件

## 容错

Flink通过**严格只处理一次的一致性保证**和**检查点与分区重新执行**来保证执行的可靠性。因为数据源是持久并可以重新获得的，如文件，持久的消息队列等，非持久数据源通过日志实现持久化。

如果发生程序故障，Flink将停止分布式流数据流。然后，系统重新启动算子，并将他们重置为最近检查点的状态。输入流也用状态快照重置，并保证作为重新启动的并行数据流中的的任何record都在所恢复检查点之后。

### 检查点

**容错机制会绘制分布式数据流和算子的状态的一致快照，包含所有算子的状态和一定时间间隔内的输入数据流。**

### Barrier

![]({{ site.url }}/assets/img/2020-12-7-12.png)

**checkpoint barrier 插入到数据流中，并和record一起流动**。barrier严格地按照直线流动并且不会超过 record。如图所示，**barrier 带有一个ID，并且将数据流中的record分为当前快照的record集(右部)和下一个快照的record集(左部)**。barrier 不会中断数据流的流动，因此非常轻量级。来自不同快照的多个barrier可以同时出现在数据流中，这意味着可以同时进行多个快照。

算子接收到barrier时进入**对齐阶段，确保接收到每个输入流中的 barrier**，才会向输出流中传递barrier。当sink(数据流DAG中的结尾)接收到barrier n时，它会向检查点协调器确认快照n。当所有的sink都接收到barrier时，快照n完成。

![]({{ site.url }}/assets/img/2020-12-7-13.png)

如图所示，当算子接收多个输入流时，需要对 barrier 进行对齐：

1. 当算子接收到某个输入流的 barrier n 后，它就**不能继续处理该数据流的后续record**，直到算子接收到其余输入流的 barrier n。否则快照n和快照n+1的record将会混淆。
2. 算子**将不能处理的record放到input buffer中**
3. 当算子接收到最后一个输入流中的 barrier n 时，算子会**向后传递所有等待的输出record以及 barrier n**
4. 经过以上步骤，算子恢复所有输入流数据的处理，优先处理input buffer中的record

### 状态

所有类型的[状态]({% post_url 2024-1-13-2-flink-state %})都是快照的一部分。

- **用户定义状态**：由transformation函数(`map()`,`filter()`)直接创建或者修改的状态
- **系统状态**：这种状态指的是数据缓存，是算子计算的一部分。例如窗口，其中缓存一定数量的record，直到计算完成为止。

![]({{ site.url }}/assets/img/2020-12-7-14.png)

如图所示，算子在从输入流接收到所有barrier之后，向输出流发出barrier之前，对其状态进行快照。在这个时间点，barrier之前的record进行的所有状态更新已经完成，并且没有依赖于barrier之后的record。由于快照状态占用空间可能很大，因此将其存储在可配置的[后端存储系统]({% post_url 2024-1-13-2-flink-state %}#存储)中。默认情况下，使用JobManager的内存，但**对于生产用途，应配置分布式可靠存储(例如HDFS)**。状态存储后，算子确认检查点，将barrier发送到输出流，然后继续处理数据。

快照包含：

- 对于并行输入数据源：快照创建时数据流中的位置偏移
- 对于算子：存储在快照中的状态的指针

### 只一次 vs. 至少一次

**对齐阶段可能会导致流处理的延迟**。通常，这种额外的延迟大约是几毫秒，但是出现过异常的延迟显着增加的情况。对于要求所有记录始终具有超低等待时间（几毫秒）的应用程序，Flink可以跳过对齐阶段。**当算子接收到一个barrier，就立即进行状态快照。算子在检查点n创建之前，会继续处理属于检查点n+1的record，这就导致检查点n与检查点n+1之间存在数据重叠**。

对于`map()`, `flatmap()`, `fliter()`等的并行操作**即使在只一次的模式中仍然会保证至少一次，因为他们没有多个输入流**。

### 异步状态快照

当快照存储在后端存储系统中时，会停止处理输入数据，这种同步操作会在每次快照创建时引入延迟。让算子在存储状态快照时继续处理输入，从而**将状态快照以后台异步进行**。为了做到这一点，**算子必须能够生成一个后续修改不影响之前状态的状态对象**。例如RocksDB中使用的写时复制类型的数据结构。

## 恢复策略

一旦遇到故障，Flink选择最近一个完成的检查点k，重置所有算子的状态到检查点k，数据源被置为从检查点k位置读取。如果是增量快照，算子需要从最新的全量快照恢复，然后对此状态进行一系列增量更新。

Flink 支持了不同级别的故障恢复策略，`jobmanager.execution.failover-strategy` 的可配置项有两种：**full 和 region**。

- Full：集群中的Task发生故障，那么**该Job的所有Task都会发生重启**。大量Task重启会导致长时间任务不能正常工作，导致数据延迟
- Region：Flink 会把Task分成不同的Region，**当某一个Task发生故障时，Flink会计算需要故障恢复的最小 Region**
  - 发生错误的Task所在的Region需要重启
  - 如果当前Region的依赖数据出现损坏或者部分丢失，那么生产数据的 Region 也需要重启
  - 为了保证数据一致性，当前Region的下游Region也需要重启

## 重启策略

如果用户配置了checkpoint，但没有设置重启策略，那么会按照固定延迟重启策略模式进行重启；如果用户没有配置checkpoint，那么默认不会重启

### 无重启策略

```
restart-strategy: none
```

```java
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.noRestart());
```

### 固定延迟重启策略

```
restart-strategy: fixed-delay
# 每个task重试
restart-strategy.fixed-delay.attempts: 3
# 每次重试间隔
restart-strategy.fixed-delay.delay: 5 s
```

```java
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(
        3, // 重启次数
        Time.of(5, TimeUnit.SECONDS) // 时间间隔
));
```

### 失败率重启策略

```
restart-strategy: failure-rate
# 时间范围内最大失败次数
restart-strategy.failure-rate.max-failures-per-interval: 3
# 时间范围
restart-strategy.failure-rate.failure-rate-interval: 5 min
# 重试间隔时间
restart-strategy.failure-rate.delay: 5 s
```

```java
env.setRestartStrategy(RestartStrategies.failureRateRestart(
        3, // 每个时间间隔的最大故障次数
        Time.of(5, TimeUnit.MINUTES), // 测量故障率的时间间隔
        Time.of(5, TimeUnit.SECONDS) //  每次任务失败时间间隔
));
```

## REFERENCE

1. CARBONE P, KATSIFODIMOS A, EWEN S, 等. Apache flink: Stream and batch processing in a single engine[J]. Bulletin of the IEEE Computer Society Technical Committee on Data Engineering, IEEE Computer Society, 2015, 36(4). 
2. [flink官方文档](https://ci.apache.org/projects/flink/flink-docs-release-1.10/)