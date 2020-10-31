---
layout: post
title:  "Spark源码阅读(二十)：计算引擎之Task与TaskContext"
date:   2020-9-25
categories: Spark
keywords: Spark SparkCore
mathjax: false
author: wzx
---

简述task的实现细节




## `TaskContext`

**维护了task执行时的上下文信息，对于每个partition都是独立的**。此抽象类只有两个实现类`TaskContextImpl`和`BarrierTaskContext`

`TaskContext`的伴生对象中定义了以下静态方法和静态属性

- `taskContext`: `ThreadLocal[TaskContext]`，保证每个task尝试线程的`TaskContext`的线程安全性

- `get()`, `set()`, `unset()`: 从`taskContext`中获得或者添加或者删除正在运行task的上下文`TaskContext`

- `getPartitionId()`: 获取当前正在运行task的`TaskContext`的partiton id

  ```scala
  def getPartitionId(): Int = {
    val tc = taskContext.get()
    if (tc eq null) {
      0
    } else {
      tc.partitionId()
    }
  }
  ```

### `TaskContextImpl`

有以下重要成员属性

- `stageId`: task所属的stage

- `stageAttemptNumber`: task所属的stage尝试次数

- `partitionId`

- `taskAttemptId`: task尝试id

- `attemptNumber`: 当前task的尝试次数

- `taskMemoryManager`

- `onCompleteCallbacks`: 保存任务执行完成后需要回调的`TaskCompletionListener`数组

- `onFailureCallbacks`: 保存任务执行失败后需要回调的`TaskFailureListener`数组

- `reasonIfKilled`: 字符串类型，记录被kill的原因

- `interrupted`: task是否被kill。之所以用interrupted作为任务尝试被kill的状态变量，是因为kill实际是通过对执行任务尝试的线程进行中断实现的

- `completed`

- `failed`

- `failure`: 导致task失败的异常

- `_fetchFailedException`

有以下重要成员方法

- `addTaskCompletionListener()`: 用于向`onCompleteCallbacks`中添加`TaskCompletionListener`，如果task已经完成则直接调用listener的回调函数

  ```scala
  @GuardedBy("this")
  override def addTaskCompletionListener(listener: TaskCompletionListener)
  : this.type = synchronized {
    if (completed) {
      listener.onTaskCompletion(this)
    } else {
      onCompleteCallbacks += listener
    }
    this
  }
  ```

- `addTaskFailureListener()`: 用于向`onFailureCallbacks`中添加`TaskFailureListener`，如果task已经失败则直接调用listener的回调函数

  ```scala
  @GuardedBy("this")
  override def addTaskFailureListener(listener: TaskFailureListener)
  : this.type = synchronized {
    if (failed) {
      listener.onTaskFailure(this, failure)
    } else {
      onFailureCallbacks += listener
    }
    this
  }
  ```

- `markTaskFailed()`, `markTaskCompleted()`: 标记task为失败或者完成并调用相关listener。如果没有标记为失败或者成功则标记并依次逆序调用对应listener的对应回调方法对当前task进行处理

  ```scala
  @GuardedBy("this")
  private[spark] override def markTaskFailed(error: Throwable): Unit = synchronized {
    if (failed) return
    failed = true
    failure = error
    invokeListeners(onFailureCallbacks, "TaskFailureListener", Option(error)) {
      _.onTaskFailure(this, error)
    }
  }

  @GuardedBy("this")
  private[spark] override def markTaskCompleted(error: Option[Throwable]): Unit = synchronized {
    if (completed) return
    completed = true
    invokeListeners(onCompleteCallbacks, "TaskCompletionListener", error) {
      _.onTaskCompletion(this)
    }
  }

  private def invokeListeners[T](
    listeners: Seq[T],
    name: String,
    error: Option[Throwable])(
    callback: T => Unit): Unit = {
    val errorMsgs = new ArrayBuffer[String](2)
    // Process callbacks in the reverse order of registration
    listeners.reverse.foreach { listener =>
      try {
        callback(listener)
      } catch {
        case e: Throwable =>
        errorMsgs += e.getMessage
        logError(s"Error in $name", e)
      }
    }
    if (errorMsgs.nonEmpty) {
      throw new TaskCompletionListenerException(errorMsgs, error)
    }
  }
  ```

- `markInterrupted()`: 标记task已经被kill

  ```scala
  private[spark] override def markInterrupted(reason: String): Unit = {
    reasonIfKilled = Some(reason)
  }
  ```

### `BarrierTaskContext`

barrier stage中的task的`TaskContext`，通过`BarrierTaskContext.get()`获得。屏障调度说明在此stage中，**设置一个全局屏障并等待所有task的所有partition到达这个屏障**。使用方法如下所示，**在一个屏障stage中，每个task必须调用相同数量次`barrier()`，否则会一直等待直至抛出timeout异常**

```scala
// 正确用法，在正式启动MPI程序前，先通过barrier操作同步等待所有Task完成磁盘写数据操作，然后通过第一个Task去拉起一个MPI job，等待MPI job执行完毕
rdd.barrier().mapPartitions { iter =>
   // Write iter to disk.
   ???
   // Fetch TaskContext
   val context = BarrierTaskContext.get()
   // Wait until all tasks finished writing.
   context.barrier()
   // The 0-th task launches an MPI job.
   if (context.partitionId() == 0) {
     val hosts = context.getTaskInfos().map(_.address)
     // Set up MPI machine file using host infos and Launch the MPI job by calling mpirun.
     ???
   }
   // Wait until the MPI job finished.
   context.barrier()
   // Collect output and return.
	 ???
}

// 错误用法，有的partition并没有调用context.barrier()
rdd.barrier().mapPartitions { iter =>
  val context = BarrierTaskContext.get()
  if (context.partitionId() == 0) {
    // Do nothing.
  } else {
    context.barrier()
  }
  iter
}

// 错误用法，第二次barrier()调用可能会导致超时
rdd.barrier().mapPartitions { iter =>
  val context = BarrierTaskContext.get()
  try {
    // Do something that might throw an Exception.
    doSomething()
    context.barrier()
  } catch {
    case e: Exception => logWarning("...", e)
  }
  context.barrier()
  iter
}
```

下面主要研究一下`barrier()`方法。**通过`barrierCoordinator`(其实是driver上的`RpcEndpointRef`)发送一条要求响应的消息，并在规定时间内(默认时间为1年!!)等待响应**

```scala
@Experimental
@Since("2.4.0")
def barrier(): Unit = {
  logInfo(s"Task $taskAttemptId from Stage $stageId(Attempt $stageAttemptNumber) has entered " +
          s"the global sync, current barrier epoch is $barrierEpoch.")
  logTrace("Current callSite: " + Utils.getCallSite())

  val startTime = System.currentTimeMillis()
  val timerTask = new TimerTask {
    override def run(): Unit = {
      logInfo(s"Task $taskAttemptId from Stage $stageId(Attempt $stageAttemptNumber) waiting " +
              s"under the global sync since $startTime, has been waiting for " +
              s"${(System.currentTimeMillis() - startTime) / 1000} seconds, current barrier epoch " +
              s"is $barrierEpoch.")
    }
  }
  // Log the update of global sync every 60 seconds.
  timer.schedule(timerTask, 60000, 60000)

  try {
    barrierCoordinator.askSync[Unit](
      message = RequestToSync(numTasks, stageId, stageAttemptNumber, taskAttemptId,
                              barrierEpoch),
      // Set a fixed timeout for RPC here, so users shall get a SparkException thrown by
      // BarrierCoordinator on timeout, instead of RPCTimeoutException from the RPC framework.
      timeout = new RpcTimeout(31536000 /* = 3600 * 24 * 365 */ seconds, "barrierTimeout"))
    barrierEpoch += 1
    logInfo(s"Task $taskAttemptId from Stage $stageId(Attempt $stageAttemptNumber) finished " +
            "global sync successfully, waited for " +
            s"${(System.currentTimeMillis() - startTime) / 1000} seconds, current barrier epoch is " +
            s"$barrierEpoch.")
  } catch {
    case e: SparkException =>
    logInfo(s"Task $taskAttemptId from Stage $stageId(Attempt $stageAttemptNumber) failed " +
            "to perform global sync, waited for " +
            s"${(System.currentTimeMillis() - startTime) / 1000} seconds, current barrier epoch " +
            s"is $barrierEpoch.")
    throw e
  } finally {
    timerTask.cancel()
    timer.purge()
  }
}

private val barrierCoordinator: RpcEndpointRef = {
  val env = SparkEnv.get
  RpcUtils.makeDriverRef("barrierSync", env.conf, env.rpcEnv)
}
```

## `Task`

task是Spark中作业运行的最小单位，为了容错，每个task可能会有一到多次task尝试。`Task`有两个子类`ShuffleMapTask`和`ResultTask`两种。**每次task尝试都会申请单独的连续内存，以执行计算**

下面是抽象类`Task`的成员属性

- `stageId`
- `stageAttemptId`
- `partitionId`
- `localProperties`: task执行所需的属性信息
- `jobId`
- `appId`
- `isBarrier`
- `context`:  `TaskContextImpl`
- `taskThread`: 运行task尝试的线程
- `_reasonIfKilled`: task被kill的原因，用于判断task是否被kill

下面是一些成员方法

- `kill()`: 标记`Task`和`TaskContextImpl`为killed，如果`interruptThread`为true则会中断运行此task的线程

- `runTask()`: 未实现的抽象方法

- `run()`: **task运行的模板方法**

  - 将task注册到`BlockInfoManager`，**创建task尝试上下文`TaskContextImpl`或者是`BarrierTaskContext`并注册到`TaskContext`中**，使用了`ThreadLocal`保证了线程安全
  - 如果task尝试已经被标记为kill，则调用`kill()`方法
  - 创建`CallerContext`
  - **调用未实现的`runTask()`方法来运行task**
  - 无论task尝试是否成功，在finally块中会调用`TaskContextImpl.markTaskCompleted()`方法，执行所有注册的`TaskCompletionListener.onTaskCompletion()`方法。
  - 在finally块中会调用`MemoryStore.releaseUnrollMemoryForThisTask()`方法，释放task尝试所占用的堆内和堆外内存，**唤醒任何等待在`MemoryManager`上的线程(`MemoryPool`中的`lock`对象就是`MemoryManager`自己)**，然后调用`TaskContext.unset()`方法，移除`ThreadLocal`中保存的当前task线程的`TaskContextImpl`

  ```scala
  final def run(
    taskAttemptId: Long,
    attemptNumber: Int,
    metricsSystem: MetricsSystem): T = {
    SparkEnv.get.blockManager.registerTask(taskAttemptId)
    val taskContext = new TaskContextImpl(
      stageId,
      stageAttemptId, // stageAttemptId and stageAttemptNumber are semantically equal
      partitionId,
      taskAttemptId,
      attemptNumber,
      taskMemoryManager,
      localProperties,
      metricsSystem,
      metrics)

    context = if (isBarrier) {
      new BarrierTaskContext(taskContext)
    } else {
      taskContext
    }

    InputFileBlockHolder.initialize()
    TaskContext.setTaskContext(context)
    taskThread = Thread.currentThread()

    if (_reasonIfKilled != null) {
      kill(interruptThread = false, _reasonIfKilled)
    }

    new CallerContext(
      "TASK",
      SparkEnv.get.conf.get(APP_CALLER_CONTEXT),
      appId,
      appAttemptId,
      jobId,
      Option(stageId),
      Option(stageAttemptId),
      Option(taskAttemptId),
      Option(attemptNumber)).setCurrentContext()

    try {
      runTask(context)
    } catch {
      case e: Throwable =>
      // Catch all errors; run task failure callbacks, and rethrow the exception.
      try {
        context.markTaskFailed(e)
      } catch {
        case t: Throwable =>
        e.addSuppressed(t)
      }
      context.markTaskCompleted(Some(e))
      throw e
    } finally {
      try {
        // Call the task completion callbacks. If "markTaskCompleted" is called twice, the second
        // one is no-op.
        context.markTaskCompleted(None)
      } finally {
        try {
          Utils.tryLogNonFatalError {
            // Release memory used by this thread for unrolling blocks
            SparkEnv.get.blockManager.memoryStore.releaseUnrollMemoryForThisTask(MemoryMode.ON_HEAP)
            SparkEnv.get.blockManager.memoryStore.releaseUnrollMemoryForThisTask(
              MemoryMode.OFF_HEAP)
            // Notify any tasks waiting for execution memory to be freed to wake up and try to
            // acquire memory again. This makes impossible the scenario where a task sleeps forever
            // because there are no other tasks left to notify it. Since this is safe to do but may
            // not be strictly necessary, we should revisit whether we can remove this in the
            // future.
            val memoryManager = SparkEnv.get.memoryManager
            memoryManager.synchronized { memoryManager.notifyAll() }
          }
        } finally {
          // Though we unset the ThreadLocal here, the context member variable itself is still
          // queried directly in the TaskRunner to check for FetchFailedExceptions.
          TaskContext.unset()
          InputFileBlockHolder.unset()
        }
      }
    }
  }
  ```

### `ShuffleMapTask`

**用于map计算的结果在Shuffle之前映射到不同的partition**。下面分析一下实现的`runTask()`方法

- 对`taskBinary`(`DAGSchuler.submitMissingTasks()`中task提交时封装的广播变量)进行反序列化，得到RDD和`ShuffleDependency`
- 调用`SortShuffleManager.getWriter()`方法，获取对指定partition的数据进行磁盘写操作的`SortShuffleWriter`
- **调用`RDD.iterator()`方法进行迭代计算**
- **调用`SortShuffleWriter.write()`方法将计算的中间结果写入磁盘文件**，等待执行完成并关闭`SortShuffleWriter`

```scala
override def runTask(context: TaskContext): MapStatus = {
  // Deserialize the RDD using the broadcast variable.
  val threadMXBean = ManagementFactory.getThreadMXBean
  val deserializeStartTime = System.currentTimeMillis()
  val deserializeStartCpuTime = if (threadMXBean.isCurrentThreadCpuTimeSupported) {
    threadMXBean.getCurrentThreadCpuTime
  } else 0L
  val ser = SparkEnv.get.closureSerializer.newInstance()
  val (rdd, dep) = ser.deserialize[(RDD[_], ShuffleDependency[_, _, _])](
    ByteBuffer.wrap(taskBinary.value), Thread.currentThread.getContextClassLoader)
  _executorDeserializeTime = System.currentTimeMillis() - deserializeStartTime
  _executorDeserializeCpuTime = if (threadMXBean.isCurrentThreadCpuTimeSupported) {
    threadMXBean.getCurrentThreadCpuTime - deserializeStartCpuTime
  } else 0L

  var writer: ShuffleWriter[Any, Any] = null
  try {
    val manager = SparkEnv.get.shuffleManager
    writer = manager.getWriter[Any, Any](dep.shuffleHandle, partitionId, context)
    writer.write(rdd.iterator(partition, context).asInstanceOf[Iterator[_ <: Product2[Any, Any]]])
    writer.stop(success = true).get
  } catch {
    case e: Exception =>
    try {
      if (writer != null) {
        writer.stop(success = false)
      }
    } catch {
      case e: Exception =>
      log.debug("Could not stop writer", e)
    }
    throw e
  }
}
```

### `ResultTask`

读取上游`ShuffleMapTask`输出的数据并计算得到最终的结果。下面分析一下实现的`runTask()`方法

- 对`taskBinary`(`DAGSchuler.submitMissingTasks()`中task提交时封装的广播变量)进行反序列化，得到RDD和计算函数`func`
- 调用`RDD.iterator()`方法进行迭代计算后**传入`func`进行action算子计算**

```scala
override def runTask(context: TaskContext): U = {
  // Deserialize the RDD and the func using the broadcast variables.
  val threadMXBean = ManagementFactory.getThreadMXBean
  val deserializeStartTime = System.currentTimeMillis()
  val deserializeStartCpuTime = if (threadMXBean.isCurrentThreadCpuTimeSupported) {
    threadMXBean.getCurrentThreadCpuTime
  } else 0L
  val ser = SparkEnv.get.closureSerializer.newInstance()
  val (rdd, func) = ser.deserialize[(RDD[T], (TaskContext, Iterator[T]) => U)](
    ByteBuffer.wrap(taskBinary.value), Thread.currentThread.getContextClassLoader)
  _executorDeserializeTime = System.currentTimeMillis() - deserializeStartTime
  _executorDeserializeCpuTime = if (threadMXBean.isCurrentThreadCpuTimeSupported) {
    threadMXBean.getCurrentThreadCpuTime - deserializeStartCpuTime
  } else 0L

  func(context, rdd.iterator(partition, context))
}
```

## REFERENCE

1. Spark内核设计的艺术：架构设计与实现

