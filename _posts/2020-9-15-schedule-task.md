---
layout: post
title:  "Spark源码阅读(十七)：调度系统之task调度"
date:   2020-9-15 9:00
categories: Spark
keywords: Spark, 调度系统
mathjax: false
author: wzx
---

介绍Spark中的`TaskSchedulerImpl`及其依赖的组件




## `TaskResultGetter`

**使用线程池对序列化的本地或远程task执行结果进行反序列化，以得到task执行结果**。有以下重要的属性

- `scheduler`: `TaskSchedulerImpl`
- `THREADS`: 获取task执行结果的线程数。通过`spark.resultGetter.threads`属性配置，默认为4
- `getTaskResultExecutor`: 固定大小为`THREADS`的线程池`Executors.newFixedThreadPool`
- `serializer`: `ThreadLocal[SerializerInstance]`，通过使用本地线程缓存，保证在使用`SerializerInstance`时是线程安全的
- `taskResultSerializer`: `ThreadLocal[SerializerInstance]`，通过使用本地线程缓存，保证在使用`SerializerInstance`对task的执行结果进行反序列化时是线程安全的

有以下两个成员方法用于处理执行成功和失败的task

- `enqueueSuccessfulTask()`: **向线程池提交反序列`serializedData`的任务，并交由`TaskSetManager`处理**

  - 向`getTaskResultExecutor`的线程池中提交以下任务
  - 对`serializedData`使用`serializer`进行反序列化，**如果是得到`DirectTaskResult`，说明task结果在本地**。首先检查获取该task结果后会不会超出`maxResultSize`，如果超过大小则kill这个task，否则用`taskResultSerializer`对task结果中保存的数据进行反序列
  - **如果得到的是`IndirectTaskResult`，说明task结果在远端**。首先检查获取该task结果后会不会超出`maxResultSize`，如果超过大小则调用`BlockManagerMaster.removeBlock()`删除远端block并kill这个task。否则调用`BlockManager.getRemoteBytes()`获取远端block数据，并使用`serializer`反序列化block数据得到`DirectTaskResult`，调用`taskResultSerializer`对task结果中保存的数据进行反序列
  - 最后调用`TaskSchedulerImpl.handleSuccessfulTask()`方法，实际上调用了`TaskSetManager.handleSuccessfulTask()`方法去处理反序列化后的task结果

  ```scala
  // TaskSetManager
  def canFetchMoreResults(size: Long): Boolean = sched.synchronized {
    totalResultSize += size
    calculatedTasks += 1
    if (maxResultSize > 0 && totalResultSize > maxResultSize) {
      val msg = s"Total size of serialized results of ${calculatedTasks} tasks " +
      s"(${Utils.bytesToString(totalResultSize)}) is bigger than spark.driver.maxResultSize " +
      s"(${Utils.bytesToString(maxResultSize)})"
      logError(msg)
      abort(msg)
      false
    } else {
      true
    }
  }

  def enqueueSuccessfulTask(
    taskSetManager: TaskSetManager,
    tid: Long,
    serializedData: ByteBuffer): Unit = {
    getTaskResultExecutor.execute(new Runnable {
      override def run(): Unit = Utils.logUncaughtExceptions {
        try {
          val (result, size) = serializer.get().deserialize[TaskResult[_]](serializedData) match {
            case directResult: DirectTaskResult[_] =>
            if (!taskSetManager.canFetchMoreResults(serializedData.limit())) {
              // kill the task so that it will not become zombie task
              scheduler.handleFailedTask(taskSetManager, tid, TaskState.KILLED, TaskKilled(
                "Tasks result size has exceeded maxResultSize"))
              return
            }
            // deserialize "value" without holding any lock so that it won't block other threads.
            // We should call it here, so that when it's called again in
            // "TaskSetManager.handleSuccessfulTask", it does not need to deserialize the value.
            directResult.value(taskResultSerializer.get())
            (directResult, serializedData.limit())
            case IndirectTaskResult(blockId, size) =>
            if (!taskSetManager.canFetchMoreResults(size)) {
              // dropped by executor if size is larger than maxResultSize
              sparkEnv.blockManager.master.removeBlock(blockId)
              // kill the task so that it will not become zombie task
              scheduler.handleFailedTask(taskSetManager, tid, TaskState.KILLED, TaskKilled(
                "Tasks result size has exceeded maxResultSize"))
              return
            }
            logDebug("Fetching indirect task result for TID %s".format(tid))
            scheduler.handleTaskGettingResult(taskSetManager, tid)
            val serializedTaskResult = sparkEnv.blockManager.getRemoteBytes(blockId)
            if (!serializedTaskResult.isDefined) {
              /* We won't be able to get the task result if the machine that ran the task failed
                   * between when the task ended and when we tried to fetch the result, or if the
                   * block manager had to flush the result. */
              scheduler.handleFailedTask(
                taskSetManager, tid, TaskState.FINISHED, TaskResultLost)
              return
            }
            val deserializedResult = serializer.get().deserialize[DirectTaskResult[_]](
              serializedTaskResult.get.toByteBuffer)
            // force deserialization of referenced value
            deserializedResult.value(taskResultSerializer.get())
            sparkEnv.blockManager.master.removeBlock(blockId)
            (deserializedResult, size)
          }

          // Set the task result size in the accumulator updates received from the executors.
          // We need to do this here on the driver because if we did this on the executors then
          // we would have to serialize the result again after updating the size.
          result.accumUpdates = result.accumUpdates.map { a =>
            if (a.name == Some(InternalAccumulator.RESULT_SIZE)) {
              val acc = a.asInstanceOf[LongAccumulator]
              assert(acc.sum == 0L, "task result size should not have been set on the executors")
              acc.setValue(size.toLong)
              acc
            } else {
              a
            }
          }

          scheduler.handleSuccessfulTask(taskSetManager, tid, result)
        } catch {
          case cnf: ClassNotFoundException =>
          val loader = Thread.currentThread.getContextClassLoader
          taskSetManager.abort("ClassNotFound with classloader: " + loader)
          // Matching NonFatal so we don't catch the ControlThrowable from the "return" above.
          case NonFatal(ex) =>
          logError("Exception while getting task result", ex)
          taskSetManager.abort("Exception while getting task result: %s".format(ex))
        }
      }
    })
  }
  ```

- `enqueueFailedTask()`: **反序列task失败原因并交由`TaskSetManager`处理**

  - 对执行结果反序列化，得到类型为`TaskFailedReason`的失败原因
  - 调用`TaskSchedulerImpl.handleFailedTask()`，实际内部调用了`TaskSetManager.handleFailedTask()`方法去处理task失败的情况

  ```scala
  def enqueueFailedTask(taskSetManager: TaskSetManager, tid: Long, taskState: TaskState,
                        serializedData: ByteBuffer) {
    var reason : TaskFailedReason = UnknownReason
    try {
      getTaskResultExecutor.execute(new Runnable {
        override def run(): Unit = Utils.logUncaughtExceptions {
          val loader = Utils.getContextOrSparkClassLoader
          try {
            if (serializedData != null && serializedData.limit() > 0) {
              reason = serializer.get().deserialize[TaskFailedReason](
                serializedData, loader)
            }
          } catch {
            case cnd: ClassNotFoundException =>
            // Log an error but keep going here -- the task failed, so not catastrophic
            // if we can't deserialize the reason.
            logError(
              "Could not deserialize TaskEndReason: ClassNotFound with classloader " + loader)
            case ex: Exception => // No-op
          } finally {
            // If there's an error while deserializing the TaskEndReason, this Runnable
            // will die. Still tell the scheduler about the task failure, to avoid a hang
            // where the scheduler thinks the task is still running.
            scheduler.handleFailedTask(taskSetManager, tid, taskState, reason)
          }
        }
      })
    } catch {
      case e: RejectedExecutionException if sparkEnv.isStopped =>
      // ignore it
    }
  }
  ```

## `LauncherBackend`

**用户应用程序使用`LauncherServer`与Spark应用程序通信**。`TaskSchedulerImpl`的底层依赖于`LauncherBackend`，`LauncherBackend`依赖于`BackendConnection`跟LauncherServer进行通信

<img src="{{ site.url }}/assets/img/2020-9-15-5.png" style="zoom:50%;" />

1. 调用`LauncherBackend.connect()`方法创建`BackendConnection`，并且创建线程执行。构造`BackendConnection`的过程中，`BackendConnection`会和LauncherServer之间建立起Socket连接，并不断从Socket连接中读取LauncherServer发送的数据
2. 调用`LauncherBackend.setAppId()`方法或`SetState()`方法，通过Socket连接向LauncherServer发送SetAppId消息或SetState消息
3. LauncherServer发送stop消息。`BackendConnection`从Socket连接中读取到LauncherServer发送的Stop消息，然后调用`LauncherBackend.fireStopRequest()`方法停止请求。

## `SchedulerBackend`

`SchedulerBackend`是`TaskScheduler`的调度后端接口。**`TaskScheduler`给task分配资源实际是通过`SchedulerBackend`来完成的，`SchedulerBackend`给task分配完资源后将与分配给task的Executor通信，并要求后者运行Task**

特质`SchedulerBackend`定义了接口规范，如下所示

- `appId`, `applicationId()`: 当前job的app id
- `start()`: 启动`SchedulerBackend`
- `stop():` 停止`SchedulerBackend`
- `reviveOffers()`: 给调度池中的所有task分配资源
- `defaultParallelism()`: 获取job的默认并行度
- `killTask()`: 向Executor请求kill指定task
- `isReady()`: `SchedulerBackend`是否准备就绪
- `getDriverLogUrls()`: 获取Driver日志的Url。这些Url将被用于在Spark UI的Executors标签页中展示

如图所示，有以下实现类

<img src="{{ site.url }}/assets/img/2020-9-15-6.png" style="zoom:67%;" />

1. `LocalSchedulerBackend`: local模式中的调度后端接口。在local模式下，Executor、`LocalSchedulerBackend`、Driver都运行在同一个JVM进程中。
2. `CoarseGrainedSchedulerBackend`: 由`CoarseGrainedSchedulerBackend`建立的`CoarseGrainedExecutorBackend`进程将会一直存在，真正的Executor线程将在`CoarseGrainedExecutorBackend`进程中执行，其子类分别代表了standalone, yarn-cluster, yarn-client, mesos等部署模式

### `LocalEndpoint`

**与`LocalSchedulerBackend`进行通信，`LocalSchedulerBackend`中保存有`LocalEndpoint`的`RpcEndpointRef`，并将命令传递给Executor**。有以下成员属性

- `userClassPath`: `Seq[URL]`。用户指定的ClassPath

- `scheduler`: Driver中的`TaskSchedulerImpl`

- `executorBackend`: 与`LocalEndpoint`相关联的`LocalSchedulerBackend`

- `totalCores`: **用于执行任务的CPU内核总数**。local模式下，totalCores固定为1

- `freeCores`: **空闲的CPU内核数**。应用程序提交的Task正式运行之前，freeCores与totalCores相等

- `localExecutorId`: local部署模式下，固定为driver

- `localExecutorHostname`: local部署模式下，固定为localhost

- `executor`: 此Executor在LocalEndpoint构造的过程中就已经实例化`new Executor(localExecutorId, localExecutorHostname, SparkEnv.get, userClassPath, isLocal = true)`

`LocalEndpoint`继承了`ThreadSafeRpcEndpoint`，重写了`receive()`和`receiveAndReply()`方法以实现消息的接收处理

- `reviveOffers()`: **给调度池中task分配资源并执行**

  - 创建`WorkerOffer`序列用于标识每个Executor上的资源
  - 调用`scheduler.resourceOffers()`获取需要分配资源的task，实际上**将task的调度交由`TaskScheduler`执行**，并在此Executor上执行task

  ```scala
  def reviveOffers() {
    val offers = IndexedSeq(new WorkerOffer(localExecutorId, localExecutorHostname, freeCores,
                                            Some(rpcEnv.address.hostPort)))
    for (task <- scheduler.resourceOffers(offers).flatten) {
      freeCores -= scheduler.CPUS_PER_TASK
      executor.launchTask(executorBackend, task)
    }
  }
  ```

- `receive()`: 处理消息

  - 如果是`ReviveOffers`消息，则会调用`reviveOffers()`给task分配资源并调度执行
  - 如果是`StatusUpdate`消息，则会调用`scheduler.statusUpdate()`更新task状态，如果已完成则回收资源并调用`receiveOffers()`
  - 如果是`KillTask`消息，则调用`executor.killTask()`kill掉task

  ```scala
  override def receive: PartialFunction[Any, Unit] = {
    case ReviveOffers =>
    reviveOffers()

    case StatusUpdate(taskId, state, serializedData) =>
    scheduler.statusUpdate(taskId, state, serializedData)
    if (TaskState.isFinished(state)) {
      freeCores += scheduler.CPUS_PER_TASK
      reviveOffers()
    }

    case KillTask(taskId, interruptThread, reason) =>
    executor.killTask(taskId, interruptThread, reason)
  }
  ```

- `receiveAndReply()`: 只处理`StopExecutor`消息，调用`executor.stop()`停止Executor并响应

### `LocalSchedulerBackend`

**`TaskSchedulerImpl`通过`LocalSchedulerBackend`与`LocalEndpoint`进行消息交互并在Executor上执行task**。下面是重要的成员属性

- `conf`: `SparkConf`
- `scheduler`: `TaskSchedulerImpl`
- `totalCores`: 固定为1
- `localEndpoint`: `LocalEndpoint`的`NettyRpcEndpointRef`
- `userClassPath`: `Seq[URL]`。用户指定的类路径。通过`spark.executor.extraClassPath`属性进行配置
- `listenerBus`: `LiveListenerBus`
- `launcherBackend`: `LauncherBackend`的匿名实现类的实例

下面是实现的重要的成员方法

- `start()`: 启动

  - 注册`LocalEndpoint`的`NettyRpcEndpointRef`
  - 向`listenerBus`投递`SparkListenerExecutorAdded`
  - 调用`LauncherBackend`向`LauncherServer`发送`SetAppId`和`SetState`消息

  ```scala
  override def start() {
    val rpcEnv = SparkEnv.get.rpcEnv
    val executorEndpoint = new LocalEndpoint(rpcEnv, userClassPath, scheduler, this, totalCores)
    localEndpoint = rpcEnv.setupEndpoint("LocalSchedulerBackendEndpoint", executorEndpoint)
    listenerBus.post(SparkListenerExecutorAdded(
      System.currentTimeMillis,
      executorEndpoint.localExecutorId,
      new ExecutorInfo(executorEndpoint.localExecutorHostname, totalCores, Map.empty)))
    launcherBackend.setAppId(appId)
    launcherBackend.setState(SparkAppHandle.State.RUNNING)
  }
  ```

- `reviveOffers()`, `killTask()`, `statusUpdate()`: 向`localEndpoint`发送对应的消息

## `TaskSchedulerImpl`

特质`TaskScheduler`，定义了task调度的接口规范，只有一个实现类`TaskSchedulerImpl`。**负责接收`DAGScheduler`给每个`Stage`创建的`TaskSet`，按照调度算法将资源分配给task，将task交给Spark集群不同节点上的Executor运行，在这些task执行失败时进行重试，通过推测执行减轻落后的task对整体作业进度的影响**。有以下重要的成员属性

- `sc`: `SparkContext`
- `maxTaskFailures`: task失败的最大次数
- `isLocal`: 是否是Local部署模式
- `conf`: `SparkConf`
- `SPECULATION_INTERVAL_MS`: **检测task是否需要推测执行的时间间隔**。通过`spark.speculation.interval`属性进行配置，默认为100ms
- `MIN_TIME_TO_SPECULATION`: task至少需要运行的时间。**task只有超过此时间限制，才允许推测执行，启动副本task**。避免对小task开启推测执行。固定为100
- `speculationScheduler`: **对task进行推测执行的线程池**，以task-scheduler-speculation为前缀，且线程池的大小为1
- `STARVATION_TIMEOUT_MS`: 判断`TaskSet`饥饿的阈值。通过`spark.starvation. timeout`属性配置，默认为15s
- `CPUS_PER_TASK`: 每个task需要分配的CPU核数。通过`spark.task.cpus`属性配置，默认为1
- `taskSetsByStageIdAndAttempt`: `HashMap[Int, HashMap[Int,TaskSetManager]]`，stage id与task尝试id和`TaskSetManager`的对应关系
- `taskIdToTaskSetManager`: task与`TaskSetManager`的映射关系。
- `taskIdToExecutorId`: task与执行此task的Executor的映射关系
- `hasReceivedTask`: 标记`TaskSchedulerImpl`是否已经接收到task
- `hasLaunchedTask`: 标记`TaskSchedulerImpl`接收的task是否已经有运行过的
- `starvationTimer`: 处理饥饿的定时器
- `nextTaskId`: `AtomicLong`，用于提交task生成task id
- `executorIdToRunningTaskIds`: `HashMap[String, HashSet[Long]]`，Executor和运行在其上的task之间的映射关系，由此看出一个Executor上可以运行多个Task
- `hostToExecutors`: `HashMap[String, HashSet[String]]`，Host(机器)与运行在此Host上的Executor之间的映射关系，由此可以看出一个Host上可以有多个Executor
- `hostsByRack`: `HashMap[String,HashSet[String]]`，机架和机架上Host(机器)之间的映射关系
- `executorIdToHost`: Executor与Executor运行所在Host(机器)的映射关系
- `backend`: 调度后端接口`SchedulerBackend`
- `mapOutputTracker`: `MapOutputTrackerMaster`
- `rootPool`: 根调度池
- `schedulingModeConf`: 调度模式名称。通过`spark.scheduler.mode`属性配置，默认为FIFO
- `schedulingMode`: 调度模式枚举类。根据`schedulingModeConf`获得

下面是重要成员方法

- `initialize()`: 初始化调度器，构建调度池

  ```scala
  def initialize(backend: SchedulerBackend) {
    this.backend = backend
    schedulableBuilder = {
      schedulingMode match {
        case SchedulingMode.FIFO =>
        new FIFOSchedulableBuilder(rootPool)
        case SchedulingMode.FAIR =>
        new FairSchedulableBuilder(rootPool, conf)
        case _ =>
        throw new IllegalArgumentException(s"Unsupported $SCHEDULER_MODE_PROPERTY: " +
                                           s"$schedulingMode")
      }
    }
    schedulableBuilder.buildPools()
  }
  ```

- `start()`: 启动调度器，如果不是在Local模式下并且开启了推测执行(`spark.speculation`, 默认为false)，则使用`speculationScheduler`设置一个执行间隔为`SPECULATION_INTERVAL_MS`的定时检查可推测执行task的定时器

  ```scala
  override def start() {
    backend.start()

    if (!isLocal && conf.getBoolean("spark.speculation", false)) {
      logInfo("Starting speculative execution thread")
      speculationScheduler.scheduleWithFixedDelay(new Runnable {
        override def run(): Unit = Utils.tryOrStopSparkContext(sc) {
          checkSpeculatableTasks()
        }
      }, SPECULATION_INTERVAL_MS, SPECULATION_INTERVAL_MS, TimeUnit.MILLISECONDS)
    }
  }
  ```

- `checkSpeculatableTasks()`: 检查可推测执行的task。实际调用了根调度池的`checkSpeculatableTasks`，如果需要推测执行则调用`SchedulerBackend.reviveOffers()`对`LocalEndpoint`发送消息，并最终分配资源以运行task

  ```scala
  def checkSpeculatableTasks() {
    var shouldRevive = false
    synchronized {
      shouldRevive = rootPool.checkSpeculatableTasks(MIN_TIME_TO_SPECULATION)
    }
    if (shouldRevive) {
      backend.reviveOffers()
    }
  }
  ```

- `submitTasks()`: 提交`TaskSet`，由`DAGScheduler`调用

  - 由传递的`TaskSet`创建`TaskSetManager`并在成员属性中注册
  - 该stage的所有已有`TaskSetManager`都标记为僵尸状态，因为stage只有一个存活的`TaskSetManager`
  - 调用`schedulableBuilder.addTaskSetManager()`**将`TaskSet`放入调度池中**
  - 如果当前应用程序不是Local模式并且没有接收到task，则调用`starvationTimer`开启一个定时器检查`hasLaunchedTask`成员变量，并不断打印warning日志警告worker没有分配足够的资源，直到有任务执行并退出
  - **调用`backend.reviveOffers()`在Executor上分配资源并运行task**

  ```scala
  private[scheduler] def createTaskSetManager(
    taskSet: TaskSet,
    maxTaskFailures: Int): TaskSetManager = {
    new TaskSetManager(this, taskSet, maxTaskFailures, blacklistTrackerOpt)
  }

  override def submitTasks(taskSet: TaskSet) {
    val tasks = taskSet.tasks
    logInfo("Adding task set " + taskSet.id + " with " + tasks.length + " tasks")
    this.synchronized {
      val manager = createTaskSetManager(taskSet, maxTaskFailures)
      val stage = taskSet.stageId
      val stageTaskSets =
      taskSetsByStageIdAndAttempt.getOrElseUpdate(stage, new HashMap[Int, TaskSetManager])

      stageTaskSets.foreach { case (_, ts) =>
        ts.isZombie = true
      }
      stageTaskSets(taskSet.stageAttemptId) = manager
      schedulableBuilder.addTaskSetManager(manager, manager.taskSet.properties)

      if (!isLocal && !hasReceivedTask) {
        starvationTimer.scheduleAtFixedRate(new TimerTask() {
          override def run() {
            if (!hasLaunchedTask) {
              logWarning("Initial job has not accepted any resources; " +
                         "check your cluster UI to ensure that workers are registered " +
                         "and have sufficient resources")
            } else {
              this.cancel()
            }
          }
        }, STARVATION_TIMEOUT_MS, STARVATION_TIMEOUT_MS)
      }
      hasReceivedTask = true
    }
    backend.reviveOffers()
  }
  ```

- `resourceOfferSingleTaskSet()`: **给单个`TaskSet`提供资源**

  - 遍历每个worker资源，如果可用cpu数大于单个task所需的cpu数，则开始分配资源
  - 调用`TaskSetManager.resourceOffer()`**由延迟调度策略返回一个task的`TaskDescription`**
  - 在成员属性中注册此task并更新`availableCpus`
  - 如果使用了屏障调度器则更新传入参数`addressesWithDescs`

  ```scala
  private def resourceOfferSingleTaskSet(
    taskSet: TaskSetManager,
    maxLocality: TaskLocality,
    shuffledOffers: Seq[WorkerOffer],
    availableCpus: Array[Int],
    tasks: IndexedSeq[ArrayBuffer[TaskDescription]],
    addressesWithDescs: ArrayBuffer[(String, TaskDescription)]) : Boolean = {
    var launchedTask = false
    // nodes and executors that are blacklisted for the entire application have already been
    // filtered out by this point
    for (i <- 0 until shuffledOffers.size) {
      val execId = shuffledOffers(i).executorId
      val host = shuffledOffers(i).host
      if (availableCpus(i) >= CPUS_PER_TASK) {
        try {
          for (task <- taskSet.resourceOffer(execId, host, maxLocality)) {
            tasks(i) += task
            val tid = task.taskId
            taskIdToTaskSetManager.put(tid, taskSet)
            taskIdToExecutorId(tid) = execId
            executorIdToRunningTaskIds(execId).add(tid)
            availableCpus(i) -= CPUS_PER_TASK
            assert(availableCpus(i) >= 0)
            // Only update hosts for a barrier task.
            if (taskSet.isBarrier) {
              // The executor address is expected to be non empty.
              addressesWithDescs += (shuffledOffers(i).address.get -> task)
            }
            launchedTask = true
          }
        } catch {
          case e: TaskNotSerializableException =>
          logError(s"Resource offer failed, task set ${taskSet.name} was not serializable")
          // Do not offer resources for this task, but don't throw an error to allow other
          // task sets to be submitted.
          return launchedTask
        }
      }
    }
    return launchedTask
  }
  ```

- `resourceOffers()`: **由集群管理器调用给task在Executor上分配资源**

  - 遍历worker资源列表，更新Host, Executor和机架的映射关系，更新黑名单列表，如果有新加入的executor则调用`TaskSetManager.executorAdded()`重新计算task的本地性
  - **对worker资源列表进行重新洗牌，避免总将task分配给一个worker**
  - 根据每个worker资源的cpu数量以生成`TaskDescription`数组的列表`tasks`，每个worker可用cpu数量列表`availableCpus`列表
  - 调用`rootPool.getSortedTaskSetQueue()`获取**按照调度算法排序好**的调度池中的所有`TaskSetManager`
  - 遍历之前排序好的`TaskSetManager`，跳过屏障调度的`TaskSetManager`并且资源不够执行的`TaskSetManager`。**按其允许的本地性等级(即`TaskSetManager.myLocalityLevels`属性)从高到低**作为参数，调用`resourceOfferSingleTaskSet()`给每个`TaskSetManager`分配资源。如果没有给任何task分配资源则做失败处理，如果使用了屏障调度并且并没有给所有task都分配资源则抛出异常并终止
  - 标记`hasLaunchedTask`已执行task并返回获得资源的task列表

  ```scala
  protected def shuffleOffers(offers: IndexedSeq[WorkerOffer]): IndexedSeq[WorkerOffer] = {
    Random.shuffle(offers)
  }

  def resourceOffers(offers: IndexedSeq[WorkerOffer]): Seq[Seq[TaskDescription]] = synchronized {
    var newExecAvail = false
    for (o <- offers) {
      if (!hostToExecutors.contains(o.host)) {
        hostToExecutors(o.host) = new HashSet[String]()
      }
      if (!executorIdToRunningTaskIds.contains(o.executorId)) {
        hostToExecutors(o.host) += o.executorId
        executorAdded(o.executorId, o.host)
        executorIdToHost(o.executorId) = o.host
        executorIdToRunningTaskIds(o.executorId) = HashSet[Long]()
        newExecAvail = true
      }
      for (rack <- getRackForHost(o.host)) {
        hostsByRack.getOrElseUpdate(rack, new HashSet[String]()) += o.host
      }
    }

    blacklistTrackerOpt.foreach(_.applyBlacklistTimeout())

    val filteredOffers = blacklistTrackerOpt.map { blacklistTracker =>
      offers.filter { offer =>
        !blacklistTracker.isNodeBlacklisted(offer.host) &&
        !blacklistTracker.isExecutorBlacklisted(offer.executorId)
      }
    }.getOrElse(offers)

    val shuffledOffers = shuffleOffers(filteredOffers)
    // Build a list of tasks to assign to each worker.
    val tasks = shuffledOffers.map(o => new ArrayBuffer[TaskDescription](o.cores / CPUS_PER_TASK))
    val availableCpus = shuffledOffers.map(o => o.cores).toArray
    val sortedTaskSets = rootPool.getSortedTaskSetQueue
    for (taskSet <- sortedTaskSets) {
      logDebug("parentName: %s, name: %s, runningTasks: %s".format(
        taskSet.parent.name, taskSet.name, taskSet.runningTasks))
      if (newExecAvail) {
        taskSet.executorAdded()
      }
    }

    for (taskSet <- sortedTaskSets) {
      val availableSlots = availableCpus.map(c => c / CPUS_PER_TASK).sum
      if (taskSet.isBarrier && availableSlots < taskSet.numTasks) {
        logInfo(s"Skip current round of resource offers for barrier stage ${taskSet.stageId} " +
                s"because the barrier taskSet requires ${taskSet.numTasks} slots, while the total " +
                s"number of available slots is $availableSlots.")
      } else {
        var launchedAnyTask = false
        // Record all the executor IDs assigned barrier tasks on.
        val addressesWithDescs = ArrayBuffer[(String, TaskDescription)]()
        for (currentMaxLocality <- taskSet.myLocalityLevels) {
          var launchedTaskAtCurrentMaxLocality = false
          do {
            launchedTaskAtCurrentMaxLocality = resourceOfferSingleTaskSet(taskSet,
                                                                          currentMaxLocality, shuffledOffers, availableCpus, tasks, addressesWithDescs)
            launchedAnyTask |= launchedTaskAtCurrentMaxLocality
          } while (launchedTaskAtCurrentMaxLocality)
        }

        if (!launchedAnyTask) {
          taskSet.getCompletelyBlacklistedTaskIfAny(hostToExecutors).foreach { taskIndex =>
            executorIdToRunningTaskIds.find(x => !isExecutorBusy(x._1)) match {
              case Some ((executorId, _)) =>
              if (!unschedulableTaskSetToExpiryTime.contains(taskSet)) {
                blacklistTrackerOpt.foreach(blt => blt.killBlacklistedIdleExecutor(executorId))

                val timeout = conf.get(config.UNSCHEDULABLE_TASKSET_TIMEOUT) * 1000
                unschedulableTaskSetToExpiryTime(taskSet) = clock.getTimeMillis() + timeout
                logInfo(s"Waiting for $timeout ms for completely "
                        + s"blacklisted task to be schedulable again before aborting $taskSet.")
                abortTimer.schedule(
                  createUnschedulableTaskSetAbortTimer(taskSet, taskIndex), timeout)
              }
              case None => // Abort Immediately
              logInfo("Cannot schedule any task because of complete blacklisting. No idle" +
                      s" executors can be found to kill. Aborting $taskSet." )
              taskSet.abortSinceCompletelyBlacklisted(taskIndex)
            }
          }
        } else {
          if (unschedulableTaskSetToExpiryTime.nonEmpty) {
            logInfo("Clearing the expiry times for all unschedulable taskSets as a task was " +
                    "recently scheduled.")
            unschedulableTaskSetToExpiryTime.clear()
          }
        }

        if (launchedAnyTask && taskSet.isBarrier) {
          if (addressesWithDescs.size != taskSet.numTasks) {
            val errorMsg =
            s"Fail resource offers for barrier stage ${taskSet.stageId} because only " +
            s"${addressesWithDescs.size} out of a total number of ${taskSet.numTasks}" +
            s" tasks got resource offers. This happens because barrier execution currently " +
            s"does not work gracefully with delay scheduling. We highly recommend you to " +
            s"disable delay scheduling by setting spark.locality.wait=0 as a workaround if " +
            s"you see this error frequently."
            logWarning(errorMsg)
            taskSet.abort(errorMsg)
            throw new SparkException(errorMsg)
          }

          // materialize the barrier coordinator.
          maybeInitBarrierCoordinator()

          // Update the taskInfos into all the barrier task properties.
          val addressesStr = addressesWithDescs
          // Addresses ordered by partitionId
          .sortBy(_._2.partitionId)
          .map(_._1)
          .mkString(",")
          addressesWithDescs.foreach(_._2.properties.setProperty("addresses", addressesStr))

          logInfo(s"Successfully scheduled all the ${addressesWithDescs.size} tasks for barrier " +
                  s"stage ${taskSet.stageId}.")
        }
      }
    }

    if (tasks.size > 0) {
      hasLaunchedTask = true
    }
    return tasks
  }
  ```

- `statusUpdate()`: 用于**更新task的状态(LAUNCHING, RUNNING, FINISHED, FAILED, KILLED, LOST)**

  - 由task id获取到对应的`TasksetManager`
  - 如果状态是LOST，调用`removeExecutor()`移除运行该task的Executor，并加入黑名单
  - 如果状态是完成状态(FINISHED, FAILED, KILLED, LOST)，则清除在调度器中的注册信息并在`TaskSetManager`中移除该运行task。如果是FINISHED状态则调用`taskResultGetter.enqueueSuccessfulTask()`对执行成功的task结果进行处理，如果是其他状态则调用`taskResultGetter.enqueueFailedTask()`对执行失败的task结果进行处理
  - 如果之前移除过Executor，则调用`dagScheduler.executorLost()`去处理移除Executor上的task，并调用`backend.reviveOffers()`给task分配资源并执行

  ```scala
  def statusUpdate(tid: Long, state: TaskState, serializedData: ByteBuffer) {
    var failedExecutor: Option[String] = None
    var reason: Option[ExecutorLossReason] = None
    synchronized {
      try {
        Option(taskIdToTaskSetManager.get(tid)) match {
          case Some(taskSet) =>
          if (state == TaskState.LOST) {
            // TaskState.LOST is only used by the deprecated Mesos fine-grained scheduling mode,
            // where each executor corresponds to a single task, so mark the executor as failed.
            val execId = taskIdToExecutorId.getOrElse(tid, {
              val errorMsg =
              "taskIdToTaskSetManager.contains(tid) <=> taskIdToExecutorId.contains(tid)"
              taskSet.abort(errorMsg)
              throw new SparkException(errorMsg)
            })
            if (executorIdToRunningTaskIds.contains(execId)) {
              reason = Some(
                SlaveLost(s"Task $tid was lost, so marking the executor as lost as well."))
              removeExecutor(execId, reason.get)
              failedExecutor = Some(execId)
            }
          }
          if (TaskState.isFinished(state)) {
            cleanupTaskState(tid)
            taskSet.removeRunningTask(tid)
            if (state == TaskState.FINISHED) {
              taskResultGetter.enqueueSuccessfulTask(taskSet, tid, serializedData)
            } else if (Set(TaskState.FAILED, TaskState.KILLED, TaskState.LOST).contains(state)) {
              taskResultGetter.enqueueFailedTask(taskSet, tid, state, serializedData)
            }
          }
          case None =>
          logError(
            ("Ignoring update with state %s for TID %s because its task set is gone (this is " +
             "likely the result of receiving duplicate task finished status updates) or its " +
             "executor has been marked as failed.")
            .format(state, tid))
        }
      } catch {
        case e: Exception => logError("Exception in statusUpdate", e)
      }
    }
    // Update the DAGScheduler without holding a lock on this, since that can deadlock
    if (failedExecutor.isDefined) {
      assert(reason.isDefined)
      dagScheduler.executorLost(failedExecutor.get, reason.get)
      backend.reviveOffers()
    }
  }
  ```


## 总结

如图所示，推测执行的一般过程

<img src="{{ site.url }}/assets/img/2020-9-15-9.png" style="zoom: 40%;" />

1. 由`speculationScheduler`线程池创建的单个线程在固定间隔时间不断调用`TaskSchedulerImpl.checkSpeculatableTasks()`
2. 实际调用了根调度池的`checkSpeculatableTasks()`方法
3. 跟调度器又会不断的递归调用直至传递给`TaskSetManager`，判断推测执行的条件并向`speculatableTasks`列表添加需要推测执行的任务。
4. 在`speculatableTasks`列表中的推测task会由`TaskSetManager.dequeueTask()`方法和`DAGSchuler`提交的task一起被取出，这部分由`TaskSchedulerImpl`进行调度。实际上`TaskSchedulerImpl`调用的是`TaskSetManager.resourceOffer()`但是其内部调用了`TaskSetManager.dequeueTask()`



如图所示，表示`TaskSchedulerImpl`的task调度流程

<img src="{{ site.url }}/assets/img/2020-9-15-7.png" style="zoom: 67%;" />

1. `DAGScheduler.submitMissingTasks()`内部调用了`TaskScheduler.submitTasks()`提交了一个`TaskSet`
2. `TaskScheduler`接收到`TaskSet`后，创建`TaskSetManager`并通过调度池构建器放入根调度池中
3. 调用`SchedulerBackend.reviveOffers()`准备在worker上分配资源以执行task
4. 第3步调用方法后，会向`RpcEndpoint`发送`ReviveOffers`消息
5. `RpcEndpoint`接收到这个消息后，调用自身的`resourceOffers()`方法，其内部调用了`TaskScheduler.resourceOffers()`传递本地的Worker资源，第6步中详述了这一过程
6. `TaskScheduler`调用根调度池的`getSortedTaskSetQueue()`方法获得按照调度算法排序后的所有`TaskSetManager`，对每个Taskset做以下操作，直到返回可以在这些Worker上执行所有task
	- 迭代可用的worker资源列表
	- 迭代调用`TaskSetManager.resourceOffer()`按照最大本地性原则和延迟调度策略选择其中的task
	- 为task创建尝试执行信息、序列化、生成TaskDescription等
7. 在`RpcEndpoint.resourceOffers()`中，已经由第6步返回了可在本地Worker上执行的task列表，调用`Executor.launchTask()`进行task尝试



如图所示，表示`TaskschedulerImpl`对task执行结果处理流程

<img src="{{ site.url }}/assets/img/2020-9-15-8.png" style="zoom:67%;" />

1. Executor在执行task的过程中，不断将task的状态通过调用`SchedulerBackend.statusUpdate()`发送更新task的状态。当task执行成功后，也会将task的完成状态通知`SchedulerBackend`
2. 在`SchedulerBackend.statusUpdate()`方法内，`SchedulerBackend`将task的状态封装为`StatusUpdate`消息通过`RpcEndpointRef`发送给`RpcEndpoint`
3. `RpcEndpoint`接收到`StatusUpdate`消息后，由`receive()`方法对消息处理，内部调用`TaskSchedulerImpl.statusUpdate()`方法，并且如果task完成会释放cpu并给新的task分配资源运行
4. 在`TaskSchedulerImpl.statusUpdate()`方法内部，如果状态为FINISHED，则调用`TaskResultGetter.enqueueSuccessfulTask()`方法
5. 在`TaskResultGetter.enqueueSuccessfulTask()`方法内部，将向线程池提交以下任务
   - 对`DirectTaskResult`类型的结果进行反序列化得到执行结果
   - 对于`IndirectTaskResult`类型的结果需要先从远端下载block数据，然后再进行反序列化得到执行结果
   - 调用`TaskSchedulerImpl.handleSuccessfulTask()`处理反序列后的执行结果
6. `TaskSchedulerImpl.handleSuccessfulTask()`方法内部实际上调用了`TaskSetManager.handleSuccessfulTask()`
7. `TaskSetManager.handleSuccessfulTask()`方法内部对执行成功的task做了必要的标记之后，就调用`DAGScheduler.taskEnded()`方法对执行结果处理。向`DAGSchedulerEventProcessLoop`投递`CompletionEvent`事件，实际上交由`DAGScheduler.handleTaskCompletion()`处理。
   - 如果是`ResultTask`，如果所有`ResultTask`都执行成功，则标记`ResultStage`为成功，并调用`JobWaiter.resultHandler()`回调函数来处理Job中每个Task的执行结果
   - 如果是`ShuffleMapTask`，调用`MapOutputTrackerMaster.registerMapOutput()`将完成信息注册。如果所有partition都执行完了，则判断`shuffleStage.isAvailable`并进行失败重试，否则标记stage成功并调用`submitWaitingChildStages()`方法处理子stage

## REFERENCE

1. Spark内核设计的艺术：架构设计与实现
