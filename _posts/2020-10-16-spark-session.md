---
layout: post
title:  "Spark源码阅读(二十五): SparkContext与SparkSession"
date:   2020-10-16
categories: Spark
keywords: Spark SparkContext SparkSession
mathjax: false
author: wzx
---

介绍Spark功能切入点`SparkContext`用来创建和操作RDD，以及一个统一的切入点`SparkSession`，封装了`SparkConf`、`SparkContext`和`SQLContext`并作为`DataSet`和`DataFrame`的切入点。




## 集群部署架构

- Cluster Manager：Spark的集群管理器，主要负责**资源的分配与管理**。集群管理器分配的资源属于一级分配，它**将各个Worker上的内存、CPU等资源分配给应用程序，但是并不负责对Executor的资源分配**。目前，**Standalone、YARN、Mesos、EC2等都可以作为Spark的集群管理器**。
- Worker：Spark的工作节点。对Spark应用程序来说，由集群管理器分配得到资源的Worker节点主要负责以下工作：**创建Executor，将资源和任务进一步分配给Executor，同步资源信息给Cluster Manager**。
- Executor：**执行计算Task的一个线进程**。主要负责Task的执行以及与Worker、Driver App的信息同步。
- Driver：客户端驱动程序，也可以理解为客户端应用程序，用于**将任务程序转换为RDD和DAG，并与Cluster Manager进行通信与调度**。

<img src="{{ site.url }}/assets/img/2020-10-16-1.png" style="zoom:50%;" />

## `SparkConf`

Spark应用程序的配置参数，**通过`CouncurrentHashMap`维护`spark.`开头的键值对参数**。一旦构造了`SparkContext`，参数就不可以改变，**Spark不支持在运行时修改参数**。官方文档列举了一些常用参数[常用参数](https://spark.apache.org/docs/2.4.6/configuration.html#application-properties)。**参数读取优先级**如下

- 优先使用`SparkConf`的setter方法设置的参数
- 通过-D传递给JVM的，或者通过`System.setProperty`设置的参数。在初始化`SparkConf`时会自动加载
- `spark-submit`传递的参数
- `spark-defaults.conf`中的参数

## `SparkContext`

**Spark功能的主要入口**，表示与Spark集群的连接，用来在集群上创建RDD，累加器和广播变量

### 包含组件

- `SparkEnv`: **Spark运行时环境**。Executor是处理任务的执行器，它依赖于`SparkEnv`提供的运行时环境。此外，在Driver中也包含了`SparkEnv`，这是为了保证local模式下任务的执行
  - `serializerManager`: **序列化管理器**
  - [`RpcEnv`: **Rpc环境**]({% post_url 2020-9-3-RpcEnv %}#rpcenv)
  - [`BlockManager`: **block管理器**]({% post_url 2020-9-9-store-blockManager %}#blockmanager)
  - [`mapOutputTracker`: **map输出跟踪器**]({% post_url 2020-9-22-schedule-mapTrack %}#mapoutputtracker)
- [`LiveListenerBus`: **`SparkContext`中的事件总线**]({% post_url 2020-7-22-listener-bus %}#listenerbus)，接收各个使用方的事件，并且通过异步方式对事件进行匹配后调用`SparkListener`的不同方法
- `SparkUI`: **Spark的用户界面**。`SparkUI`间接依赖于计算引擎、调度系统、存储体系，Job、Stage、存储、Executor等组件的监控数据都会以`SparkListenerEvent`的形式投递到`LiveListenerBus`中，`SparkUI`将从各个`SparkListener`中读取数据并显示到Web界面
- `SparkStatusTracker`: **提供对Job、Stage等的监控信息**
- `ConsoleProgressBar`: 依赖于`SparkStatusTracker`，**在控制台展示Stage的进度**
- [`DAGScheduler`: **DAG调度器**]({% post_url 2020-9-14-schedule-stage %}#dagscheduler)，负责创建Job，将DAG中的RDD划分到不同的Stage、提交Stage等
- [`TaskScheduler`: **任务调度器**]({% post_url 2020-9-15-schedule-task %}#taskschedulerimpl)。`TaskScheduler`按照调度算法对集群管理器已经分配给应用程序的资源进行二次调度后分配给任务
- `HeartbeatReceiver`: **心跳接收器**。所有Executor都会向`HeartbeatReceiver`发送心跳信息，`HeartbeatReceiver`接收到Executor的心跳信息后，首先更新Executor的最后可见时间，然后将此信息交给`TaskScheduler`作进一步处理。
- `ContextCleaner`: **上下文清理器**。`ContextCleaner`实际用异步方式清理那些超出应用作用域范围的`RDD`、`ShuffleDependency`和`Broadcast`等信息
- `JobProgressListener`: **作业进度监听器**。`JobProgressListener`将注册到`LiveListenerBus`中作为事件监听器之一使用
- `EventLoggingListener`: **将事件持久化到存储的监听器**，是SparkContext中的可选组件。当`spark.eventLog.enabled`属性为true时启用。
- `ExecutorAllocationManager`: **Executor动态分配管理器**。根据工作负载动态调整Executor的数量。在配置`spark.dynamicAllocation.enabled`属性为true的前提下，在非local模式下或者当`spark.dynamicAllocation.testing`属性为true时启用
- `ShutdownHookManager`: **用于设置关闭钩子的管理器**。可以给应用设置关闭钩子，这样就可以在JVM进程退出时，执行一些清理工作

### 初始化步骤
1. 创建Spark执行环境`SparkEnv`
2. 创建RDD清理器`metadataCleaner`
3. 创建并初始化Spark UI
4. Hadoop相关配置及Executor环境变量的设置；
5. 创建任务调度`TaskScheduler`
6. 创建和启动`DAGScheduler`
7. `TaskScheduler`的启动
8. 初始化块管理器`BlockManager`
9. 启动测量系统`MetricsSystem`
10. 创建和启动Executor分配管理器`ExecutorAllocationManager`
11. `ContextCleaner`的创建与启动
12. Spark环境更新
13. 创建`DAGSchedulerSource`和`BlockManagerSource`
14. 将`SparkContext`标记为激活

## `SparkSession`

**提供了一个统一的切入点来使用Spark的各项功能，是对`SparkContext`、`SQLContext`及`DataFrame`等的一层封装**。有以下成员属性

- `sparkContext`
- `sharedState`: `SharedState`。在多个`SparkSession`之间共享的状态，包括`SparkContext`、缓存的数据、监听器及与外部系统交互的字典信息
- `sessionState`: `SessionState`。`SparkSession`的独立状态，包括SQL配置，临时表，已注册函数等
- `sqlContext`: `SQLContext`。Spark SQL的上下文信息
- `conf`: `RuntimeConfig`
- `sqlListener`: 伴生对象中的静态属性`AtomicReference[SQLListener]`，主要用于SQL UI
- `activeThreadSession`: 伴生对象中的静态属性`InheritableThreadLocal[SparkSession]`，用于**持有当前线程的激活的`SparkSession`**
- `defaultSession`: 伴生对象中的静态属性`AtomicReference[SparkSession]`，用于**持有默认的`Spark-Session`**

### `Builder`

**内部类静态类`Builder`是`SparkSession`实例的构建器**，应用了工厂方法。有以下成员变量

- `options`: `HashMap[String, String]`。用于**缓存构建`SparkConf`所需的属性配置**
- `extensions`: `SparkSessionExtensions`。用于拓展Spark Catalyst中的拓展规则
- `userSuppliedContext`: `Option[SparkContext]`。用于**持有用户提供的`SparkContext`**

有以下成员方法

- `config()`, `appName()`, `master()`, `enableHiveSupport()`: 向`options`属性中添加配置参数**。

- `getOrCreate()`: **获取或创建`SparkSession`**

  - 判断是否在driver上，只有在driver上才可能创建`SparkSession`
  - 如果能从`activeThreadSession`中获取到`SparkSession`并且还未停止，则直接返回。否则使用`SparkSession.synchronized`同步以下操作
  - 尝试返回未停止的`defaultSession`
  - 尝试获取`userSuppliedContext`中的`SparkContext`，或则使用`options`内的配置属性创建一个`SparkContext`
  - 加载Catalyst拓展规则
  - 构建`SparkSession`并放入`activeThreadSession`和`defaultSession`并返回

  ```scala
  def getOrCreate(): SparkSession = synchronized {
  assertOnDriver()
    // Get the session from current thread's active session.
    var session = activeThreadSession.get()
    if ((session ne null) && !session.sparkContext.isStopped) {
      applyModifiableSettings(session)
      return session
    }

    // Global synchronization so we will only set the default session once.
  SparkSession.synchronized {
      // If the current thread does not have an active session, get it from the global session.
      session = defaultSession.get()
      if ((session ne null) && !session.sparkContext.isStopped) {
        applyModifiableSettings(session)
      return session
      }

      // No active nor global default session. Create a new one.
      val sparkContext = userSuppliedContext.getOrElse {
      val sparkConf = new SparkConf()
        options.foreach { case (k, v) => sparkConf.set(k, v) }

        // set a random app name if not given.
      if (!sparkConf.contains("spark.app.name")) {
          sparkConf.setAppName(java.util.UUID.randomUUID().toString)
        }

        SparkContext.getOrCreate(sparkConf)
        // Do not update `SparkConf` for existing `SparkContext`, as it's shared by all sessions.
      }

      // Initialize extensions if the user has defined a configurator class.
      val extensionConfOption = sparkContext.conf.get(StaticSQLConf.SPARK_SESSION_EXTENSIONS)
      if (extensionConfOption.isDefined) {
        val extensionConfClassName = extensionConfOption.get
        try {
          val extensionConfClass = Utils.classForName(extensionConfClassName)
          val extensionConf = extensionConfClass.newInstance()
          .asInstanceOf[SparkSessionExtensions => Unit]
          extensionConf(extensions)
        } catch {
        // Ignore the error if we cannot find the class or when the class has the wrong type.
          case e @ (_: ClassCastException |
                    _: ClassNotFoundException |
                    _: NoClassDefFoundError) =>
          logWarning(s"Cannot use $extensionConfClassName to configure session extensions.", e)
      }
      }

      session = new SparkSession(sparkContext, None, None, extensions)
      options.foreach { case (k, v) => session.initialSessionOptions.put(k, v) }
      setDefaultSession(session)
      setActiveSession(session)

      // Register a successfully instantiated context to the singleton. This should be at the
      // end of the class definition so that the singleton is updated only if there is no
    // exception in the construction of the instance.
      sparkContext.addSparkListener(new SparkListener {
        override def onApplicationEnd(applicationEnd: SparkListenerApplicationEnd): Unit = {
          defaultSession.set(null)
        }
      })
    }

    return session
  }
  ```

## REFERENCE

1. Spark内核设计的艺术：架构设计与实现

