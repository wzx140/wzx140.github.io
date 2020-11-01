---
layout: post
title:  "Spark源码阅读(十二)：存储体系之Broadcast"
date:   2020-9-13
categories: Spark
keywords: Spark, 存储体系
mathjax: false
author: wzx
---

介绍Spark中的广播实现方式




## `BroadcastManager`

用于将配置信息和序列化后的RDD、Job及ShuffleDependency等信息在本地存储。

有以下重要的成员变量

- `initialized`：`BroadcastManager`是否完成初始化
- `isDriver`
- `nextBroadcastId`: 下一个广播对象的广播ID，`AtomicLong`保证原子操作
- `broadcastFactory`: 广播工厂实例
- `cachedValues`: `ReferenceMap(AbstractReferenceMap.HARD, AbstractReferenceMap.WEAK)`。保存所有`Broadcast`的值，key是强引用(`BroacastId`)，value是虚引用(广播对象)

以下是重要方法

- `initialize()`: **初始化广播工厂**，在使用广播前由`SparkContext`或者`Executor`调用。在没有初始化的情况下，先实例化`TorrentBroadcastFactory`作为`broadcastFactory`，在调用`broadcastFactory.initialize()`对工厂进行初始化，最后标记已经完成初始化。整个方法使用`synchronized`保证线程安全，不会重复初始化。

  ```scala
  private def initialize() {
    synchronized {
      if (!initialized) {
        broadcastFactory = new TorrentBroadcastFactory
        broadcastFactory.initialize(isDriver, conf, securityManager)
        initialized = true
      }
    }
  }
  ```

- `stop()`: 代理`broadcastFactory.stop()`

  ```scala
  def stop() {
    broadcastFactory.stop()
  }

  // TorrentBroadcastFactory
  override def stop() { }
  ```

- `newBroadcast()`: 代理`broadcastFactory.newBroadcast[T]()`，`nextBroadcastId`自增，构造新的`Broadcast`

  ```scala
  def newBroadcast[T: ClassTag](value_ : T, isLocal: Boolean): Broadcast[T] = {
    broadcastFactory.newBroadcast[T](value_, isLocal, nextBroadcastId.getAndIncrement())
  }

  // TorrentBroadcastFactory
  override def newBroadcast[T: ClassTag](value_ : T, isLocal: Boolean, id: Long): Broadcast[T] = {
    new TorrentBroadcast[T](value_, id)
  }
  ```

- `unbroadcast()`: 代理`broadcastFactory.unbroadcast()`

  ```scala
  def unbroadcast(id: Long, removeFromDriver: Boolean, blocking: Boolean) {
    broadcastFactory.unbroadcast(id, removeFromDriver, blocking)
  }

  // TorrentBroadcastFactory
  override def unbroadcast(id: Long, removeFromDriver: Boolean, blocking: Boolean) {
    TorrentBroadcast.unpersist(id, removeFromDriver, blocking)
  }
  ```

## `Broadcast`

只有唯一子类`TorrentBroadcast`，当被构造时，会调用`writeBlocks()`将构造参数中的`value`写入本地

有以下重要成员变量

- `compressionCodec`: 压缩编码解码器

- `_value`: 从Executor或者Driver上读取的广播的值，通过`readBroadcastBlock()`方法读取，懒加载

- `blockSize`: block大小，只读属性，通过`spark.broadcast.blockSize`设置，默认为4MB

- `broadcastId`: case class

  ```scala
  @DeveloperApi
  case class BroadcastBlockId(broadcastId: Long, field: String = "") extends BlockId {
    override def name: String = "broadcast_" + broadcastId + (if (field == "") "" else "_" + field)
  }
  ```

- `numBlocks`: 广播变量包含的block数量，不可变，通过`writeBlocks()`获取
- `checksumEnabled`: 是否开启检验和，通过`spark.broadcast.checksum`配置，默认为true
- `checksums`: 储存所有block的检验和数组

下面是重要方法

- `writeBlocks()`: **将对象在本地保存两份，一份是未序列的，一份分成多个block并序列化，放入本地存储体系中**。

  - `BlockManager.putSingle()`方法将广播对象写入本地的存储体系。当Spark以local模式运行时，则会将广播对象写入Driver本地的存储体系。
  - `TorrentBroadcast.blockifyObject()`方法将对象转化为一系列被压缩和序列化后的block
  - 生成校验和数组
  - 将广播包装成`ChunkedByteBuffer`，并为每个块生成`BroadcastBlockId`类型的pieceId，调用`BlockManager.putBytes()`放入本地存储系统，这部分用于远端来读取的

  ```scala
  private def writeBlocks(value: T): Int = {
    import StorageLevel._
    // Store a copy of the broadcast variable in the driver so that tasks run on the driver
    // do not create a duplicate copy of the broadcast variable's value.
    val blockManager = SparkEnv.get.blockManager
    if (!blockManager.putSingle(broadcastId, value, MEMORY_AND_DISK, tellMaster = false)) {
      throw new SparkException(s"Failed to store $broadcastId in BlockManager")
    }
    val blocks =
    TorrentBroadcast.blockifyObject(value, blockSize, SparkEnv.get.serializer, compressionCodec)
    if (checksumEnabled) {
      checksums = new Array[Int](blocks.length)
    }
    blocks.zipWithIndex.foreach { case (block, i) =>
      if (checksumEnabled) {
        checksums(i) = calcChecksum(block)
      }
      val pieceId = BroadcastBlockId(id, "piece" + i)
      val bytes = new ChunkedByteBuffer(block.duplicate())
      if (!blockManager.putBytes(pieceId, bytes, MEMORY_AND_DISK_SER, tellMaster = true)) {
        throw new SparkException(s"Failed to store $pieceId of $broadcastId in local BlockManager")
      }
    }
    blocks.length
  }
  ```

- `readBroadcastBlock()`: **`_value`需要时才会调用这个方法，获取广播block**

  - 根据`broadcastId`从`cachedValues`中获取数据，如果没有则由以下步骤获取
  - 调用`BlockManager.getLocalValues()`从本地存储系统中获取广播对象
  - 如果获得，释放当前`broadcastId`的锁，放入`broadcastCache`缓存
  - 如果没有获得，说明数据是通过`BlockManager.putBytes()`方法以序列化方式写入存储体系的。调用`readBlocks()`获取广播block，调用`unBlockifyObject()`方法将广播block转换为广播对象并返回。调用`BlockManager.putSingle()`方法将广播对象写入本地存储并放入`broadcastCache`缓存

  ```scala
  private def readBroadcastBlock(): T = Utils.tryOrIOException {
    TorrentBroadcast.synchronized {
      val broadcastCache = SparkEnv.get.broadcastManager.cachedValues

      Option(broadcastCache.get(broadcastId)).map(_.asInstanceOf[T]).getOrElse {
        setConf(SparkEnv.get.conf)
        val blockManager = SparkEnv.get.blockManager
        blockManager.getLocalValues(broadcastId) match {
          case Some(blockResult) =>
          if (blockResult.data.hasNext) {
            val x = blockResult.data.next().asInstanceOf[T]
            releaseLock(broadcastId)

            if (x != null) {
              broadcastCache.put(broadcastId, x)
            }

            x
          } else {
            throw new SparkException(s"Failed to get locally stored broadcast data: $broadcastId")
          }
          case None =>
          logInfo("Started reading broadcast variable " + id)
          val startTimeMs = System.currentTimeMillis()
          val blocks = readBlocks()
          logInfo("Reading broadcast variable " + id + " took" + Utils.getUsedTimeMs(startTimeMs))

          try {
            val obj = TorrentBroadcast.unBlockifyObject[T](
              blocks.map(_.toInputStream()), SparkEnv.get.serializer, compressionCodec)
            // Store the merged copy in BlockManager so other tasks on this executor don't
            // need to re-fetch it.
            val storageLevel = StorageLevel.MEMORY_AND_DISK
            if (!blockManager.putSingle(broadcastId, obj, storageLevel, tellMaster = false)) {
              throw new SparkException(s"Failed to store $broadcastId in BlockManager")
            }

            if (obj != null) {
              broadcastCache.put(broadcastId, obj)
            }

            obj
          } finally {
            blocks.foreach(_.dispose())
          }
        }
      }
    }
  }
  ```

- `readBlocks()`: **用于读取已经序列化和压缩后的广播block**

  - 新建数组用于存放广播块，对广播block进行随机洗牌，**避免对广播block的读取出现热点**
  - 调用`BlockManager.getLocalBytes()`方法根据构造的案例类`BroadcastBlockId`从本地获取广播块。
  - 如果获取到，放入数组，释放当前广播块的锁
  - 如果没有获取到，调用`BlockManager.getRemoteBytes()` 方法根据构造的案例类`BroadcastBlockId`从远端获取广播block，进行校验和，调用`BlockManager.putBytes()`放入本地存储体系

  ```scala
  private def readBlocks(): Array[BlockData] = {
    // Fetch chunks of data. Note that all these chunks are stored in the BlockManager and reported
    // to the driver, so other executors can pull these chunks from this executor as well.
    val blocks = new Array[BlockData](numBlocks)
    val bm = SparkEnv.get.blockManager

    for (pid <- Random.shuffle(Seq.range(0, numBlocks))) {
      val pieceId = BroadcastBlockId(id, "piece" + pid)
      logDebug(s"Reading piece $pieceId of $broadcastId")
      // First try getLocalBytes because there is a chance that previous attempts to fetch the
      // broadcast blocks have already fetched some of the blocks. In that case, some blocks
      // would be available locally (on this executor).
      bm.getLocalBytes(pieceId) match {
        case Some(block) =>
        blocks(pid) = block
        releaseLock(pieceId)
        case None =>
        bm.getRemoteBytes(pieceId) match {
          case Some(b) =>
          if (checksumEnabled) {
            val sum = calcChecksum(b.chunks(0))
            if (sum != checksums(pid)) {
              throw new SparkException(s"corrupt remote block $pieceId of $broadcastId:" +
                                       s" $sum != ${checksums(pid)}")
            }
          }
          // We found the block from remote executors/driver's BlockManager, so put the block
          // in this executor's BlockManager.
          if (!bm.putBytes(pieceId, b, StorageLevel.MEMORY_AND_DISK_SER, tellMaster = true)) {
            throw new SparkException(
              s"Failed to store $pieceId of $broadcastId in local BlockManager")
          }
          blocks(pid) = new ByteBufferBlockData(b, true)
          case None =>
          throw new SparkException(s"Failed to get $pieceId of $broadcastId")
        }
      }
    }
    blocks
  }
  ```

- `unpersist()`: 调用`BlockManagerMaster.removeBroadcast()`方法

  ```scala
  def unpersist(id: Long, removeFromDriver: Boolean, blocking: Boolean): Unit = {
    logDebug(s"Unpersisting TorrentBroadcast $id")
    SparkEnv.get.blockManager.master.removeBroadcast(id, removeFromDriver, blocking)
  }
  ```

## 总结

广播对象写入过程

<img src="{{ site.url }}/assets/img/2020-9-13-1.png" style="zoom:50%;" />

广播对象读取过程

<img src="{{ site.url }}//assets/img/2020-9-13-2.png" style="zoom:70%;" />

## REFERENCE

1. Spark内核设计的艺术：架构设计与实现
