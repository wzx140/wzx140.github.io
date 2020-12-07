---
layout: post
title:  "Flink: 状态"
date:   2020-12-7 10:00
categories: Flink
keywords: Flink State
mathjax: true
author: wzx
---

Flink中高效而丰富的算子状态管理机制



## 状态

Flink 的状态都是基于本地的，即每个算子子任务维护着这个算子子任务对应的状态存储，**算子子任务之间的状态不能相互访问**。

### Keyed and Operator State

![]({{ site.url }}/assets/img/2020-12-7-9.png)

Keyed State以键值对方式存储，并且**与数据流一样严格分区**。如图所示，只有在`keyBy`函数之后，才可以在对应keyed stream的子任务中访问到Keyed State。这种key的对齐保证了**状态更新是本地运算**，从而保证了一致性且没有事务开销。

Operator State绑定在子任务上，**流入相同子任务的数据可以访问和共享Operator State**。


### Raw and Managed State

**Managed State用Flink runtime维护的数据结构表示**，例如内部哈希表或RocksDB。Managed State如`ValueState`、`ListState`、`MapState`等。Flink runtime对state进行编码，然后将其写入检查点。

Raw State由算子维护state的数据结构表示，Flink本身并不知道状态的数据结构。快照时，仅将字节序列写入检查点。**Raw State的数据结构对Flink不透明，只能观察到原始的字节数组**。

## 访问

### State
![]({{ site.url }}/assets/img/2020-12-7-10.png)

### StateDescriptor
![]({{ site.url }}/assets/img/2020-12-7-11.png)

### 示例

```java
public class Main {

  /**
   * (1,4)
   * (1,6)
   */
  public static void main(String[] args) throws Exception {
    StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
    env.fromElements(Tuple2.of(1, 3), Tuple2.of(1, 5), Tuple2.of(1, 7), Tuple2.of(1, 5), Tuple2.of(1, 2))
            .keyBy(x->x.f0)
            .flatMap(new CountWindowAverage())
            .print();
    env.execute("test");
  }

  private static class CountWindowAverage extends RichFlatMapFunction<Tuple2<Integer, Integer>, Tuple2<Integer, Integer>> {

    private ValueState<Tuple2<Integer, Integer>> state = null;

    @Override
    public void open(Configuration parameters) {
      ValueStateDescriptor<Tuple2<Integer, Integer>> descriptor =
              new ValueStateDescriptor<>("average", TypeInformation.of(new TypeHint<Tuple2<Integer, Integer>>() {
              }));

      // 设置ttl
      StateTtlConfig ttlConfig = StateTtlConfig.newBuilder(Time.seconds(10))
              // 何时更新过期状态
              .setUpdateType(StateTtlConfig.UpdateType.OnCreateAndWrite)
              // 是否可以访问过期数据
              .setStateVisibility(StateTtlConfig.StateVisibility.ReturnExpiredIfNotCleanedUp)
              .build();
      descriptor.enableTimeToLive(ttlConfig);

      state = getRuntimeContext().getState(descriptor);
    }

    @Override
    public void flatMap(Tuple2<Integer, Integer> value, Collector<Tuple2<Integer, Integer>> out) throws Exception {
      Tuple2<Integer, Integer> curSum = Optional
              .ofNullable(state.value())
              .orElse(Tuple2.of(0, 0));

      curSum.f0 += 1;
      curSum.f1 += value.f1;
      state.update(curSum);

      if (curSum.f0 >= 2) {
        out.collect(Tuple2.of(value.f0, curSum.f1 / curSum.f0));
        state.clear();
      }
    }
  }
}
```

## 存储

算子状态存储在JVM 的堆内存，堆外内存或者第三方存储中

### MemoryStateBackend
**`MemoryStateBackend`将状态数据存储在本地内存中**，一般用来进行本地调试用。`StreamExecutionEnvironment.setStateBackend(new MemoryStateBackend(DEFAULT_MAX_STATE_SIZE, false))`

- 每个独立状态默认限制大小为 5MB，可以通过构造函数增加容量
- 状态的大小不能超过 akka 的 Framesize 大小
- 聚合后的状态必须能够放进 JobManager 的内存中

### FsStateBackend
`FsStateBackend`会把状态数据保存在 TaskManager 的内存中。CheckPoint 时，将状态快照写入到配置的文件系统(HDFS)中，少量的元数据信息存储到 JobManager 的内存中。**适用于大作业、状态较大、全局高可用的那些任务**。`StreamExecutionEnvironment.setStateBackend(new FsStateBackend("hdfs://namenode:40010/flink/checkpoints", false))`

### RocksDBStateBackend
`RocksDBStateBackend`将正在运行中的状态数据保存在RocksDB数据库中，RocksDB 数据库默认将数据存储在 TaskManager 运行节点的数据目录下，CheckPoint 时，将状态快照写入到配置的文件系统(HDFS)中，**是唯一支持增量快照的状态后端**

## REFERENCE

1. CARBONE P, KATSIFODIMOS A, EWEN S, 等. Apache flink: Stream and batch processing in a single engine[J]. Bulletin of the IEEE Computer Society Technical Committee on Data Engineering, IEEE Computer Society, 2015, 36(4). 
2. [flink官方文档](https://ci.apache.org/projects/flink/flink-docs-release-1.10/)