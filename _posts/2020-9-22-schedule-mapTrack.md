---
layout: post
title:  "Spark源码阅读(十八)：调度系统之map输出跟踪器"
date:   2020-9-22
categories: Spark
keywords: Spark, 调度系统
mathjax: false
author: wzx
---

`MapOutputTracker`用于跟踪map任务的输出状态，此状态便于reduce任务定位map输出结果所在的节点地址，进而获取中间输出结果




## `MapStatus`

**保存`ShuffleMapTask`返回给调度器的结果**，该特质有以下未实现的方法

- `location`: `BlockManagerId`，**task运行的位置**即结果所在位置
- `getSizeForBlock()`: 返回shuffle中间数据中，**指定reduce id在此map任务中所依赖的数据大小**，单位为字节

有实现类`CompressedMapStatus`, `HighlyCompressedMapStatus`

## `ShuffleStatus`

**帮助`MapOutputTrackerMaster`记录单个`ShuffleMapStage`的状态**

有以下成员属性

- `numPartitions`
- `mapStatuses`: `Array[MapStatus](numPartitions)`。如果未计算完则对应partition中为null
- `cachedSerializedStatuses`: **缓存shuffleId与序列化`MapStatus`的映射关系**
- `cachedSerializedBroadcast`: 与`cachedSerializedStatuses`一致的广播变量，当序列化`MapStatuses`太大而无法在单个RPC中发送时应通过广播变量获取，**此变量保存广播变量的引用防止被GC**
- `_numAvailableOutputs`: 跟踪已完成的partition总数

有以下成员方法

- `addMapOutput()`: 向`mapStatuses`注册一个map输出，并调用`invalidateSerializedMapOutputStatusCache()`清除缓存

- `removeMapOutput()`: 向`mapStatuses`移除一个map输出，并调用`invalidateSerializedMapOutputStatusCache()`清除缓存

- `removeOutputsByFilter()`: 移除在`mapStatuses`中满足过滤器的map输出，并调用`invalidateSerializedMapOutputStatusCache()`清除缓存

- `removeOutputsOnHost()`, `removeOutputsOnExecutor()`: 清除指定Host或者Executor上的map输出

- `invalidateSerializedMapOutputStatusCache()`: 清除`cachedSerializedBroadcast`广播变量

  ```scala
  def invalidateSerializedMapOutputStatusCache(): Unit = synchronized {
    if (cachedSerializedBroadcast != null) {
      Utils.tryLogNonFatalError {
        // Use `blocking = false` so that this operation doesn't hang while trying to send cleanup
        // RPCs to dead executors.
        cachedSerializedBroadcast.destroy(blocking = false)
      }
      cachedSerializedBroadcast = null
    }
    cachedSerializedMapStatus = null
  }
  ```

- `findMissingPartitions()`: 返回未计算完的partition id序列
- `serializedMapStatus()`: **优先返回已经缓存`cachedSerializedStatuses`**，如果缓存为空则调用`MapOutputTracker.serializeMapStatuses()`对`mapStatuses`序列化后返回并放入`cachedSerializedStatuses`，如果长度大于`minBroadcastSize`还会将序列化的数据放入广播变量`cachedSerializedBroadcast`

## `MapOutputTracker`

抽象类，**跟踪stage的map输出的位置**，有两个实现类`MapOutputTrackerMaster`和`MapOutputTrackerWorker`

下面是一些重要成员属性

- `trackerEndpoint`: 持有Driver上`MapOutputTrackerMasterEndpoint`的`RpcEndpointRef`
- `mapStatuses`: `Map[Int, Array[MapStatus]]`**维护shuffle id与其各个map task的输出状态**。由于各个`MapOutputTrackerWorker`会向`MapOutputTrackerMaster`不断汇报map任务的状态信息，因此`MapOutputTrackerMaster`的`mapStatuses`中维护的信息是最新最全的。`MapOutputTrackerWorker`的`mapStatuses`对于本节点Executor运行的map任务状态是及时更新的，而对于其他节点上的map任务状态则更像一个缓存，在`mapStatuses`不能命中时会向Driver上的`MapOutputTrackerMaster`获取最新的任务状态信息
- `fetching`: `HashSet[Int]`shuffle id集合，用来记录**当前Executor正在从哪些map输出的位置拉取数据**

下面是一些实现的方法

- `askTracker()`: 向`MapOutputTrackerMasterEndpoint`发送消息，并期望在超时时间之内得到回复
- `sendTracker()`: 向`MapOutputTrackerMasterEndpoint`发送消息，并期望在超时时间之内获得的返回值为true

### `MapOutputTrackerMaster`

在Spark中的变量名`mapOutputTracker`一般指的是`MapOutputTrackerMaster`。**`MapOutputTrackerMaster`负责整理和维护由`MapOutputTrackerWorker`发送的所有的map任务的输出跟踪信息，只存在于Driver上**。



下面是一些重要的成员属性

- `SHUFFLE_PREF_MAP_THRESHOLD`, `SHUFFLE_PREF_REDUCE_THRESHOLD`: 当map或者reduce task的数量超过这个限制将不会分配偏好位置，因为这样做更昂贵。默认为1000

- `REDUCER_PREF_LOCS_FRACTION`:  当某个map task输出占比超过这个比例后，增大这个比例以实现本地读取。默认为0.2

- `minSizeForBroadcast`: 用于广播的最小大小，使用广播变量将map输出信息传递给Executor。通过`spark.shuffle.mapOutput.minSizeForBroadcast`属性配置，默认为512KB。`minSizeForBroadcast必`须小于`maxRpcMessageSize`

- `shuffleLocalityEnabled`: 是否为reduce任务计算本地性偏好。通过`spark.shuffle.reduceLocality.enabled`属性进行配置，默认为true

- `shuffleStatuses`: `ConcurrentHashMap[Int, ShuffleStatus]`**维护了shuffle id和`ShuffleStatus`的映射关系**

- `maxRpcMessageSize`: 最大的Rpc消息的大小。通过`spark.rpc.message.maxSize`属性进行配置，默认为128MB。`minSizeForBroadcast`必须小于`maxRpcMessageSize`

- `mapOutputRequests`: `LinkedBlockingQueue[GetMapOutputMessage]`**缓存获取map输出状态的请求消息**

- `threadpool`: 用于获取map输出的固定大小的线程池。此线程池提交的线程都以后台线程运行，且线程名以map-output-dispatcher为前缀，线程池大小可以使用`spark.shuffle.mapOutput.dispatcher.numThreads`属性配置，默认大小为8

  - 在`MessageLoop`中，循环取出阻塞队列`mapOutputRequests`中的`GetMapOutputMessage`直至遇到`PoisonPill`
  - 根据消息中的shuffle id在`shuffleStatuses`取出`shuffleStatus`，调用`serializedMapStatus()` 对`shuffleStatus`中的`mapStatuses`进行序列化并返回

  ```scala
  private val threadpool: ThreadPoolExecutor = {
    val numThreads = conf.getInt("spark.shuffle.mapOutput.dispatcher.numThreads", 8)
    val pool = ThreadUtils.newDaemonFixedThreadPool(numThreads, "map-output-dispatcher")
    for (i <- 0 until numThreads) {
      pool.execute(new MessageLoop)
    }
    pool
  }

  private class MessageLoop extends Runnable {
    override def run(): Unit = {
      try {
        while (true) {
          try {
            val data = mapOutputRequests.take()
            if (data == PoisonPill) {
              // Put PoisonPill back so that other MessageLoops can see it.
              mapOutputRequests.offer(PoisonPill)
              return
            }
            val context = data.context
            val shuffleId = data.shuffleId
            val hostPort = context.senderAddress.hostPort
            logDebug("Handling request to send map output locations for shuffle " + shuffleId +
                     " to " + hostPort)
            val shuffleStatus = shuffleStatuses.get(shuffleId).head
            context.reply(
              shuffleStatus.serializedMapStatus(broadcastManager, isLocal, minSizeForBroadcast))
          } catch {
            case NonFatal(e) => logError(e.getMessage, e)
          }
        }
      } catch {
        case ie: InterruptedException => // exit
      }
    }
  }
  ```

- `PoisonPill`: 毒药`GetMapOutputMessage`

有以下重要的成员方法

- `post()`: 向阻塞队列`mapOutputRequests`中添加消息交由`MessageLoop`线程去处理，主要由`MapOutputTrackerMasterEndpoint`接收消息并调用

  ```scala
  // MapOutputTrackerMaster
  def post(message: GetMapOutputMessage): Unit = {
    mapOutputRequests.offer(message)
  }

  /** RpcEndpoint class for MapOutputTrackerMaster */
  private[spark] class MapOutputTrackerMasterEndpoint(
      override val rpcEnv: RpcEnv, tracker: MapOutputTrackerMaster, conf: SparkConf)
    extends RpcEndpoint with Logging {

    logDebug("init") // force eager creation of logger

    override def receiveAndReply(context: RpcCallContext): PartialFunction[Any, Unit] = {
      case GetMapOutputStatuses(shuffleId: Int) =>
        val hostPort = context.senderAddress.hostPort
        logInfo("Asked to send map output locations for shuffle " + shuffleId + " to " + hostPort)
        val mapOutputStatuses = tracker.post(new GetMapOutputMessage(shuffleId, context))

      case StopMapOutputTracker =>
        logInfo("MapOutputTrackerMasterEndpoint stopped!")
        context.reply(true)
        stop()
    }
  }
  ```

- `registerShuffle()`: 注册shuffle id

  ```scala
  def registerShuffle(shuffleId: Int, numMaps: Int) {
    if (shuffleStatuses.put(shuffleId, new ShuffleStatus(numMaps)).isDefined) {
      throw new IllegalArgumentException("Shuffle ID " + shuffleId + " registered twice")
    }
  }
  ```

- `registerMapOutput()`: 在shuffle中注册`MapStatus`

  ```scala
  def registerMapOutput(shuffleId: Int, mapId: Int, status: MapStatus) {
    shuffleStatuses(shuffleId).addMapOutput(mapId, status)
  }
  ```

- `unregisterMapOutput()`, `unregisterAllMapOutput()`, `unregisterAllMapOutput()`, `unregisterShuffle()`, `removeOutputsOnHost()`, `removeOutputsOnExecutor()`: 注销对应的信息

- `getMapSizesByExecutorId()`: 获取某个shuffle中，reducer partition所要获取的map输出的对应的block序列(一个reducer需要读取多个map端的block)

- `getPreferredLocationsForShuffle()`: 获取某个shuffle中，reducer partition所要获取的map输出数据占比超过`REDUCER_PREF_LOCS_FRACTION`的Executor列表，这将作为此reducer partition的偏好位置

### `MapOutputTrackerWorker`

Executor端维护map输出信息

有以下成员属性

- `mapStatuses`: `ConcurrentHashMap[Int, Array[MapStatus]]`。维护本地map输出状态
- `fetching`: `HashSet[Int]`。缓存已经请求过的shuffle id

下面是重要的成员方法

- `getStatus()`: **根据shuffleId获取map状态信息数组**

  - 首先尝试返回本地缓存`mapStatuses`中对应的map状态信息
  - 如果没有则判断`fetching`中有没有该shuffle id
  - 如果有，说明有其他线程正在请求map状态信息，此时等待fetching的对象锁并再次从`mapStatuses`本地缓存中获取
  - 如果没有，直接调用`askTracker()`方法向`MapOutputTrackerWorker`发送`GetMapOutputStatuses`请求消息获取对应shuffle的map状态信息。放入本地缓存，从`fetching`中移除并唤醒等待在此对象锁的线程

  ```scala
  private def getStatuses(shuffleId: Int): Array[MapStatus] = {
    val statuses = mapStatuses.get(shuffleId).orNull
    if (statuses == null) {
      logInfo("Don't have map outputs for shuffle " + shuffleId + ", fetching them")
      val startTime = System.currentTimeMillis
      var fetchedStatuses: Array[MapStatus] = null
      fetching.synchronized {
        // Someone else is fetching it; wait for them to be done
        while (fetching.contains(shuffleId)) {
          try {
            fetching.wait()
          } catch {
            case e: InterruptedException =>
          }
        }

        // Either while we waited the fetch happened successfully, or
        // someone fetched it in between the get and the fetching.synchronized.
        fetchedStatuses = mapStatuses.get(shuffleId).orNull
        if (fetchedStatuses == null) {
          // We have to do the fetch, get others to wait for us.
          fetching += shuffleId
        }
      }

      if (fetchedStatuses == null) {
        // We won the race to fetch the statuses; do so
        logInfo("Doing the fetch; tracker endpoint = " + trackerEndpoint)
        // This try-finally prevents hangs due to timeouts:
        try {
          val fetchedBytes = askTracker[Array[Byte]](GetMapOutputStatuses(shuffleId))
          fetchedStatuses = MapOutputTracker.deserializeMapStatuses(fetchedBytes)
          logInfo("Got the output locations")
          mapStatuses.put(shuffleId, fetchedStatuses)
        } finally {
          fetching.synchronized {
            fetching -= shuffleId
            fetching.notifyAll()
          }
        }
      }
      logDebug(s"Fetching map output statuses for shuffle $shuffleId took " +
               s"${System.currentTimeMillis - startTime} ms")

      if (fetchedStatuses != null) {
        fetchedStatuses
      } else {
        logError("Missing all output locations for shuffle " + shuffleId)
        throw new MetadataFetchFailedException(
          shuffleId, -1, "Missing all output locations for shuffle " + shuffleId)
      }
    } else {
      statuses
    }
  }
  ```

- `getMapSizesByExecutorId()`: 调用了`getStatus()`，效果与`MapOutputTrackerWorker.getMapSizesByExecutorId()`类似

## 总结

如果所示，这是`MapOutTrackerMaster`应答获取shuffle信息的流程

![]({{ site.url }}/assets/img/2020-9-22-1.png)

1. `MapOutputTrackerMasterEndpoint`接收到`MapOutputTrackerWorker.getStatuses()`发送的`GetMapOutputStatuses`请求
2. 于是调用`MapOutputTrackerMaster.post()`将请求消息放入`mapOutputRequests`中
3. `MapOutputTrackerMaster`中的线程池的`MessageLoop`线程会循环从`mapOutputRequests`阻塞队列中取出消息
   - 由消息中的shuffle id，在`shuffleStatuses`中找到对应的`ShuffleStatus`并调用`ShuffleStatus.serializedMapStatus()`方法获取序列化后的`MapStatuses`
   - 首先从缓存的`cachedSerializedStatuses`返回
   - 如果没有则将原始的`MapStatuses`序列化后放入缓存并返回
   - 调用`context.reply()`回调方法将序列化后的数据恢复消息请求者



如图所示，这是在`MapOutputTrackerMaster`中的shuffle注册流程

<img src="{{ site.url }}/assets/img/2020-9-22-2.png" style="zoom: 33%;" />

1. `DAGScheduler`在创建了`ShuffleMapStage`后，调用`MapOutputTrackerMaster.registerShuffle()`方法向`shuffleStatuses`缓存注册shuffle id
2. `DAGScheduler`处理`ShuffleMapTask`的执行结果时，如果发现`ShuffleMapTask`所属的`ShuffleMapStage`中每一个partition的`ShuffleMapTask`都执行成功了，那么将调用`MapOutputTrackerMaster.registerMapOutputs()`方法，将`ShuffleMapStage`中每一个`ShuffleMapTask的MapStatus`保存到对应的`ShuffleStatuse`中

## REFERENCE

1. Spark内核设计的艺术：架构设计与实现
