---
layout: post
title:  "Spark作业gc时间过长"
date:   2020-5-19
categories: Spark
keywords: Spark, GC
mathjax: false
author: wzx
---

在运行spark作业时，发现GC时间非常长，基本上快占task time 的三分之一了，最后通过GC日志确定了问题所在。



## 问题
通过`spark-submit`提交到yarn集群上执行一个Spark Streaming流处理作业，通过Sparrk的web ui查看作业时，发现 Excutors下的Task Time都已经变红了。如下图所示，在GC时间超过Task Time的10%时就会出现红色警示。

![]({{ site.url }}/assets/img/2020-5-19-1.png)

## GC日志
### 选项
可以通过在JVM添加参数来输出GC日志，在`spark-submit`通过 `spark.driver.extraJavaOptions` 和 `spark.executor.extraJavaOptions` 指定

```shell
# 输出简要GC日志
-XX:+PrintGC
# 输出详细GC日志
-XX:+PrintGCDetails
# 输出GC日志到文件
-Xloggc:gc.log
# 输出GC的时间戳（以JVM启动到当期的总时长的时间戳形式）
-XX:+PrintGCTimeStamps
# 输出GC的时间戳（以日期的形式，如 2013-05-04T21:53:59.234+0800）
-XX:+PrintGCDateStamps
# 在进行GC的前后打印出堆的信息
-XX:+PrintHeapAtGC
# 打印年轻代各个引用的数量以及时长
-XX:+PrintReferenceGC
```
### 格式
以下是我的GC日志

```shell
2020-05-15T15:58:48.502+0800: 49.508: [GC (System.gc()) [PSYoungGen: 359152K->93559K(1223168K)] 376068K->110482K(4019712K), 0.0471985 secs] [Times: user=0.11 sys=0.10, real=0.05 secs]
2020-05-15T15:58:48.549+0800: 49.556: [Full GC (System.gc()) [PSYoungGen: 93559K->0K(1223168K)] [ParOldGen: 16923K->97733K(2796544K)] 110482K->97733K(4019712K), [Metaspace: 30597K->30593K(1077248K)], 0.0805504 secs] [Times: user=0.16 sys=0.05, real=0.08 secs]
2020-05-15T15:58:48.630+0800: 49.636: Total time for which application threads were stopped: 0.1281609 seconds, Stopping threads took: 0.0000755 seconds
2020-05-15T15:58:48.632+0800: 49.638: [GC (System.gc()) [PSYoungGen: 10485K->160K(1223168K)] 108219K->97893K(4019712K), 0.0018068 secs] [Times: user=0.01 sys=0.00, real=0.01 secs]
2020-05-15T15:58:48.634+0800: 49.640: [Full GC (System.gc()) [PSYoungGen: 160K->0K(1223168K)] [ParOldGen: 97733K->94577K(2796544K)] 97893K->94577K(4019712K), [Metaspace: 30598K->30598K(1077248K)], 0.0932750 secs] [Times: user=0.28 sys=0.00, real=0.09 secs]
.......
2020-05-15T15:58:50.071+0800: 51.077: [GC (System.gc()) [PSYoungGen: 181766K->10716K(1223168K)] 276343K->105301K(4019712K), 0.0083237 secs] [Times: user=0.03 sys=0.00, real=0.01 secs]
2020-05-15T15:58:50.079+0800: 51.086: [Full GC (System.gc()) [PSYoungGen: 10716K->0K(1223168K)] [ParOldGen: 94585K->100385K(2796544K)] 105301K->100385K(4019712K), [Metaspace: 40727K->40712K(1085440K)], 0.1144704 secs] [Times: user=0.34 sys=0.00, real=0.12 secs]
......
2020-05-15T15:58:50.658+0800: 51.664: [GC (System.gc()) [PSYoungGen: 92815K->17580K(1223168K)] 193200K->117973K(4019712K), 0.0092596 secs] [Times: user=0.03 sys=0.00, real=0.01 secs]
2020-05-15T15:58:50.667+0800: 51.673: [Full GC (System.gc()) [PSYoungGen: 17580K->0K(1223168K)] [ParOldGen: 100393K->110457K(2796544K)] 117973K->110457K(4019712K), [Metaspace: 41207K->41203K(1085440K)], 0.1100946 secs] [Times: user=0.33 sys=0.01, real=0.11 secs]
```

以第一条日志为例

- 2020-05-15T15:58:48.502+0800：表示时间与时区

- 49.508：表示从JVM启动到打印GC的时间间隔(秒)

- PSYoungGen: 359152K->93559K(1223168K)：表示年轻代从359152K减少到了93559K，总大小为1223168K

- 376068K->110482K(4019712K)：表示整个堆的GC情况

- 0.0471985 secs：表示整个GC时间

- [Times: user=0.11 sys=0.10, real=0.05 secs]：表示用户态占用时长，内核用时，真实用时

### 分析

这里我把一些minor GC的日志去掉了，它们的GC时间都很短，重点在这些Full GC上。可以看出，大部分Full GC都是由`System.gc()`触发的，并且每次无论是年轻代还是老年代，空间占用都很小完全不满足GC的条件，所以我们可以断定一定是程序中存在不当的显式调用`System.gc()`的情况。

我在自己的项目搜索了，发现`System.gc()`没有主动调用过，所以一定是在调用的某个外部库的代码里。这里我在JVM选项上加上`-XX:+DisableExplicitGC`后彻底解决了这个问题。

如果想要看到`System.gc()`的调用者，可以使用[async-profiler](https://github.com/jvm-profiling-tools/async-profiler/)。使用`profiler.sh start -e java.lang.System.gc <pid>`以及`profiler.sh stop -o traces <pid>`，在这之间如果调用了`System.gc()`，控制台就会打印出调用者的堆栈。

## REFERENCE

[1] [SRE 高延迟问题的罪魁祸首 System.gc()](https://www.infoq.cn/article/lXTRgYb9ecVBu*72fT7O)

[2] [JVM GC 日志详解](https://juejin.im/post/5c80b0f451882532cd57b541)

[3] [who call System.gc() in spark streaming program](https://stackoverflow.com/questions/61849214/how-can-i-know-who-call-system-gc-in-spark-streaming-program)

