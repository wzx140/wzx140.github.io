---
layout: post
title: "Flink 剖析(四) Window、Time、Watermark"
date:   2024-1-13
categories: Flink
keywords: Flink, Window, Time, Watermark
mathjax: false
author: wzx
---

Flink 通过 Window、Time、Watermark 完成乱序事件处理，定义窗口以及维护和更新用户定义状态等




## Window
流式计算最终的目的是去统计数据产生汇总结果的，而**在无界数据集上，如果做一个全局的窗口统计是不现实的，所以只能去划定一定大小的窗口范围去汇总**

![]({{ site.url }}/assets/img/2024-1-13-4-1.png)

- 滚动窗口(Tumbling Window): 窗口数据有固定的大小(时间，计数)，窗口不会重叠
- 滑动窗口(Sliding Window): 窗口数据有固定的大小(时间)，有生成间隔，窗口会重叠
- 会话窗口(Session Window): 窗口数据没有固定的大小，根据会话参数划分，窗口不会重叠

窗口是时间驱动(如每30s)或者事件驱动(如每100个元素)的。触发流程如下所示

![]({{ site.url }}/assets/img/2024-1-13-4-2.png)

- WindowAssigner：根据 record 的维度进行 keyBy，并创建 key 为 window 的 keyed state
- Trigger：触发窗口计算的条件
- evictor：触发计算后，先过滤一部分 record

Flink 对一些聚合类的窗口计算(如 sum 和 min)做了优化，因为聚合类的计算不需要将窗口中的所有数据都保存下来，只需要保存一个中间结果值就可以了。

以下代码描述了建立一个`GlobalWindow`，当窗口积累了1000个事件时，保留最新的100个并触发计算。

```java
stream
  .window(GlobalWindow.create())
  .trigger(Count.of(1000))
  .evict(Count.of(100))
```

## Time

通过`env.setStreamTimeCharacteristic()`可以设置使用时间

- **事件时间(Event Time)**: 事件实际发生的时间，由数据生产方标记
    - 需要对每条记录都保存事件时间，成本较高
    - 能处理无序事件时性能和延迟受到影响
- **摄入时间(Ingestion Time)**: 事件进入流处理框架的时间。
    - 处于 Event Time 和 Processing Time之间，**性能和准确度的折中方案**
    - 比起 Event Time，Ingestion Time可以**不需要设置复杂的 Watermark**，因此也不需要太多缓存，延迟较低。**不能保证生产者到 source 的这段线路的有序性**，因此不能处理无序事件和延迟数据
    - 相对于 Processing Time，能**保证执行图内部有序**，计算结果相对准确
- **处理时间(Processing Time)**: 事件被处理的时间
    - 只依赖当前执行机器的系统时钟，**无需缓存，最佳的性能和最低的延迟**
    - 容易受到各种因素影响(event 产生的速度、在算子之间传输速度等)

![]({{ site.url }}/assets/img/2024-1-13-4-3.png)

## WaterMark
WaterMark 本质上是一个时间戳，是 DataStream 中一个带有时间戳的元素，**一般结合事件时间使用，解决实时计算中的数据乱序问题**。

- WaterMark 是判断迟到数据的标准，同时也是窗口触发的标记。如果出现了一个 WaterMark(T)，那么就意味着 EventTime < T 的数据都已经到达
- 在程序并行度大于 1 的情况下，会有多个流产生 WaterMark 和窗口，这时候 **Flink 会选取时间戳最小的 WaterMark**
- **WaterMark 可以加在 source 算子或者非 source 算子，建议加在source算子上**

![]({{ site.url }}/assets/img/2024-1-13-4-4.png)

如上图所示，对于有序的数据流来说，WaterMark 周期性的出现在数据流中；对于无序的数据流来说，WaterMark 表示该点之前，所有到特定事件时间的事件都应该到达。一旦 WaterMark 到达算子，算子就可以将其内部事件时钟提前到 WaterMark 的值。

![]({{ site.url }}/assets/img/2024-1-13-4-5.png)

如图所示，WaterMark 由 source 直接生成，**每个算子的 subtask 都维护自己的 WaterMark**。当算子接收到 WaterMark，它会将内部事件时钟提前到WaterMark 的时间，并且为后序算子生成新的 WaterMark。**当算子从多个输入流获得 WaterMark (如 keyBy, partition 等算子)时，算子会选择 WaterMark 中的最小值更新时间**

### WatermarkStrategy
如果不是 Kafka 或者是 Kinesis，那么需要用 `WatermarkStrategy` 来指定 `TimestampAssigner`
```java
WatermarkStrategy.forMonotonousTimestamps()
    .withTimestampAssigner((event, timestamp) -> event.time);
```

Flink中已经实现 `WatermarkGenerator` 接口的预制周期性 Watermark 生成器
- **单调增加的生成策略**: `WatermarkStrategy.forMonotonousTimestamps()`
- **固定延迟的生成策略(基于事件时间)**: `WatermarkStrategy.forBoundedOutOfOrderness(Duration.ofSeconds(10))`

### Idle WaterMark

如果 source 的一个分区在某段时间内没有数据，那么这个分区产生的WaterMark 值将停滞，由于 Flink 取时间戳最小的 WaterMark，那么会**导致下游整体的 WaterMark 停滞**

```java
WatermarkStrategy
    .forBoundedOutOfOrderness(Duration.ofSeconds(20))
    // 超过1min没有数据, 生成的Watermark会携带idle标记, 不会阻碍下游的运行
    .withIdleness(Duration.ofMinutes(1));
```

### WatermarkGenerators

如果需要自定义 `WatermarkStrategy`，需要继承 `WatermarkStrategy` 并实现 `createWatermarkGenerator()` 方法，`TimestampAssigner` 对象可以在构建时传递，所以不需要实现 `createTimestampAssigner()` 方法。

```java
public interface WatermarkStrategy<T> extends TimestampAssignerSupplier<T>, WatermarkGeneratorSupplier<T>{

    /**
     * Instantiates a {@link TimestampAssigner} for assigning timestamps according to this
     * strategy.
     */
    @Override
    TimestampAssigner<T> createTimestampAssigner(TimestampAssignerSupplier.Context context);

    /**
     * Instantiates a WatermarkGenerator that generates watermarks according to this strategy.
     */
    @Override
    WatermarkGenerator<T> createWatermarkGenerator(WatermarkGeneratorSupplier.Context context);
}
```

对于 `WatermarkGenerators` 来说，需要继承并实现以下方法
```java
@Public
public interface WatermarkGenerator<T> {

    /**
     * 为每个事件调用，用于检查并记住事件时间戳，或基于事件本身产生Watermark
     */
    void onEvent(T event, long eventTimestamp, WatermarkOutput output);

    /**
     * 由ExecutionConfig.getAutoWatermarkInterval()时间间隔调用，产生Watermark
     */
    void onPeriodicEmit(WatermarkOutput output);
}
```

#### 周期性 WatermarkGenerator
通过 `ExecutionConfig.setAutoWatermarkInterval()` 设置时间间隔

```java
/**
 * 固定延迟的生成器(基于事件时间)
 */
public class BoundedOutOfOrdernessGenerator implements WatermarkGenerator<MyEvent> {

    private final long maxOutOfOrderness = 3500; // 3.5 seconds

    private long currentMaxTimestamp;

    @Override
    public void onEvent(MyEvent event, long eventTimestamp, WatermarkOutput output) {
        currentMaxTimestamp = Math.max(currentMaxTimestamp, eventTimestamp);
    }

    @Override
    public void onPeriodicEmit(WatermarkOutput output) {
        // emit the watermark as current highest timestamp minus the out-of-orderness bound
        output.emitWatermark(new Watermark(currentMaxTimestamp - maxOutOfOrderness - 1));
    }

}

/**
 * 固定延迟的生成器(基于系统时间)
 */
public class TimeLagWatermarkGenerator implements WatermarkGenerator<MyEvent> {

    private final long maxTimeLag = 5000; // 5 seconds

    @Override
    public void onEvent(MyEvent event, long eventTimestamp, WatermarkOutput output) {
        // don't need to do anything because we work on processing time
    }

    @Override
    public void onPeriodicEmit(WatermarkOutput output) {
        output.emitWatermark(new Watermark(System.currentTimeMillis() - maxTimeLag));
    }
}
```

#### Punctuated WatermarkGenerator
```java
public class PunctuatedAssigner implements WatermarkGenerator<MyEvent> {

    @Override
    public void onEvent(MyEvent event, long eventTimestamp, WatermarkOutput output) {
        if (event.hasWatermarkMarker()) {
            output.emitWatermark(new Watermark(event.getWatermarkTimestamp()));
        }
    }

    @Override
    public void onPeriodicEmit(WatermarkOutput output) {
        // don't need to do anything because we emit in reaction to events above
    }
}
```


## REFERENCE
1. CARBONE P, KATSIFODIMOS A, EWEN S, 等. Apache flink: Stream and batch processing in a single engine[J]. Bulletin of the IEEE Computer Society Technical Committee on Data Engineering, IEEE Computer Society, 2015, 36(4). 
2. [flink官方文档](https://ci.apache.org/projects/flink/flink-docs-release-1.10/)