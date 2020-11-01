---
layout: post
title:  "Spark源码阅读(十三)：调度系统总览"
date:   2020-9-14 8:00
categories: Spark
keywords: Spark, 调度系统
mathjax: false
author: wzx
---

介绍Spark中的调度系统




## 整体介绍
<img src="{{ site.url }}/assets/img/2020-9-14-1.png" style="zoom:67%;" />

如图所示，调度系统主要由`DAGScheduler`和`TaskScheduler`构成。

1. Spark调度系统对用户提交的Job，**根据RDD之间的依赖关系构建DAG**
2. `DAGScheduler`负责接收由RDD构成的DAG，**将一系列RDD划分到不同的Stage**。根据Stage的类型`ResultStage`和`ShuffleMapStage`，给Stage中未完成的Partition创建不同类型的Task `ResultTask`和`ShuffleMapTask`。`DAGScheduler`将**每个Stage中的task以`TaskSet`的形式提交给`TaskScheduler`继续处理**
3. `TaskScheduler`负责从`DAGScheduler`接收`TaskSet`，**创建`TaskSetManager`对`TaskSet`进行管理，并将此`TaskSetManager`添加到调度池中**，最后将对Task的调度交给`SchedulerBackend`处理。`SchedulerBackend`首先申请`TaskScheduler`，按**照Task调度算法对调度池中的所有`TaskSetManager`进行排序，然后对`TaskSet`按照最大本地性原则分配资源，最后在各个分配的节点上运行`TaskSet`中的Task**。使用集群管理器分配资源与任务调度，对于失败的任务还会有一定的重试与容错机制
4. 执行任务，并将任务中间结果和最终结果存入存储体系

## REFERENCE

1. Spark内核设计的艺术：架构设计与实现
