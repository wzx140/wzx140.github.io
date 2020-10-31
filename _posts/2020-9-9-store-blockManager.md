---
layout: post
title:  "Spark源码阅读(十)：存储体系之block管理与调度"
date:   2020-9-9 9:00
categories: Spark
keywords: Spark SparkCore
mathjax: false
author: wzx
---

介绍Spark中的`BlockManager`和`BlockManagerMaster`




## `BlockManager`

对之前介绍的那些存储体系的组件进行封装，**提供对本地或远端节点上的磁盘及堆内堆外内存中block的管理**。

- `shuffleClient`: shuffle客户端。用于和`NettyBlockTransferService`进行交互，以便当前节点将block上传到其他节点或者从其他节点下载block到本地。如果`spark.shuffle.service.enabled`为true(默认false)则使用`ExternalShuffleClient`替代`blockTransferService`

```scala
private[spark] val shuffleClient = if (externalShuffleServiceEnabled) {
  val transConf = SparkTransportConf.fromSparkConf(conf, "shuffle", numUsableCores)
  new ExternalShuffleClient(transConf, securityManager,
                            securityManager.isAuthenticationEnabled(), conf.get(config.SHUFFLE_REGISTRATION_TIMEOUT))
} else {
  blockTransferService
}
```

`BlockManager`中的成员变量基本是前几章所述的对象，这里只分析一些重要的方法。

- `initialize()`: 初始化

  - 初始化`blockTransferService`, `shuffleClient`
  - 由`spark.storage.replication.policy`指定block复制策略，默认是`RandomBlockReplicationPolicy`
  - 生成`BlockManagerId`, `shuffleServerId`
  - 当启用了外部Shuffle服务，并且当前BlockManager所在节点不是Driver时，需要注册外部的Shuffle服务

  ```scala
  def initialize(appId: String): Unit = {
    blockTransferService.init(this)
    shuffleClient.init(appId)

    blockReplicationPolicy = {
      val priorityClass = conf.get(
        "spark.storage.replication.policy", classOf[RandomBlockReplicationPolicy].getName)
      val clazz = Utils.classForName(priorityClass)
      val ret = clazz.newInstance.asInstanceOf[BlockReplicationPolicy]
      logInfo(s"Using $priorityClass for block replication policy")
      ret
    }

    val id =
    BlockManagerId(executorId, blockTransferService.hostName, blockTransferService.port, None)

    val idFromMaster = master.registerBlockManager(
      id,
      maxOnHeapMemory,
      maxOffHeapMemory,
      slaveEndpoint)

    blockManagerId = if (idFromMaster != null) idFromMaster else id

    shuffleServerId = if (externalShuffleServiceEnabled) {
      logInfo(s"external shuffle service port = $externalShuffleServicePort")
      BlockManagerId(executorId, blockTransferService.hostName, externalShuffleServicePort)
    } else {
      blockManagerId
    }

    // Register Executors' configuration with the local shuffle service, if one should exist.
    if (externalShuffleServiceEnabled && !blockManagerId.isDriver) {
      registerWithExternalShuffleServer()
    }

    logInfo(s"Initialized BlockManager: $blockManagerId")
  }
  ```

- `getCurrentBlockStatus()`: **获取目前最新的block信息**，将block的状态信息封装为`BlockStatus`，增加内存占用大小和磁盘占用大小信息(一个block不可能部分在磁盘上部分在内存上)，同时更新了一下`StorageLevel`(在`putBytes()`中调用因为存在内存保存不下从而储存在磁盘中)

  ```scala
  private def getCurrentBlockStatus(blockId: BlockId, info: BlockInfo): BlockStatus = {
    info.synchronized {
      info.level match {
        case null =>
        BlockStatus.empty
        case level =>
        val inMem = level.useMemory && memoryStore.contains(blockId)
        val onDisk = level.useDisk && diskStore.contains(blockId)
        val deserialized = if (inMem) level.deserialized else false
        val replication = if (inMem  || onDisk) level.replication else 1
        val storageLevel = StorageLevel(
          useDisk = onDisk,
          useMemory = inMem,
          useOffHeap = level.useOffHeap,
          deserialized = deserialized,
          replication = replication)
        val memSize = if (inMem) memoryStore.getSize(blockId) else 0L
        val diskSize = if (onDisk) diskStore.getSize(blockId) else 0L
        BlockStatus(storageLevel, memSize, diskSize)
      }
    }
  }
  ```

- `getStatus()`: 与`getCurrentBlockStatus()`功能一样，只是没有更新`StorageLevel`

- `reportAllBlocks()`: 遍历所有block，调用`BlockManagerMaster.updateBlockInfo()`方法**报告所有block信息**

  ```scala
  private def tryToReportBlockStatus(
    blockId: BlockId,
    status: BlockStatus,
    droppedMemorySize: Long = 0L): Boolean = {
    val storageLevel = status.storageLevel
    val inMemSize = Math.max(status.memSize, droppedMemorySize)
    val onDiskSize = status.diskSize
    master.updateBlockInfo(blockManagerId, blockId, storageLevel, inMemSize, onDiskSize)
  }

  private def reportAllBlocks(): Unit = {
    logInfo(s"Reporting ${blockInfoManager.size} blocks to the master.")
    for ((blockId, info) <- blockInfoManager.entries) {
      val status = getCurrentBlockStatus(blockId, info)
      if (info.tellMaster && !tryToReportBlockStatus(blockId, status)) {
        logError(s"Failed to report $blockId to master; giving up.")
        return
      }
    }
  }
  ```

- `reportBlockStatus()`: **向`BlockManagerMaster`报告block信息**。如果没有注册则调用`asyncReregister()`方法另起线程调用`reregister()`在`BlockManagerMaster`中注册

  ```scala
  private def reportBlockStatus(
    blockId: BlockId,
    status: BlockStatus,
    droppedMemorySize: Long = 0L): Unit = {
    val needReregister = !tryToReportBlockStatus(blockId, status, droppedMemorySize)
    if (needReregister) {
      logInfo(s"Got told to re-register updating block $blockId")
      // Re-registering will report our new block for free.
      asyncReregister()
    }
    logDebug(s"Told master about block $blockId")
  }
  ```

- `reregister()`: 向`BlockManagerMaster`重新注册`BlockManager`并报告所有block信息

  ```scala
  def reregister(): Unit = {
    logInfo(s"BlockManager $blockManagerId re-registering with master")
    master.registerBlockManager(blockManagerId, maxOnHeapMemory, maxOffHeapMemory, slaveEndpoint)
    reportAllBlocks()
  }
  ```

- `getLocalBytes()`: **从本地存储体系中获得封装为`BlockData`的序列化的block**

  - 由`BlockInfoManager`获得block的读锁
  - 由`BlockInfo`确定block的元信息，确定由`MemoryStore`和`DiskStore`获取。
  - **如果存储等级是未序列化，则优先从磁盘中读取以省去序列化的操作**，否则从内存中读取未序列化的block进行序列化
  - 如果存储等级是序列化，则优先从内存中读取block，否则从硬盘中读取block(序列化)并**尝试将该block放入内存中**。

  ```scala
  private def doGetLocalBytes(blockId: BlockId, info: BlockInfo): BlockData = {
    val level = info.level
    logDebug(s"Level for block $blockId is $level")
    // In order, try to read the serialized bytes from memory, then from disk, then fall back to
    // serializing in-memory objects, and, finally, throw an exception if the block does not exist.
    if (level.deserialized) {
      // Try to avoid expensive serialization by reading a pre-serialized copy from disk:
      if (level.useDisk && diskStore.contains(blockId)) {
        // Note: we purposely do not try to put the block back into memory here. Since this branch
        // handles deserialized blocks, this block may only be cached in memory as objects, not
        // serialized bytes. Because the caller only requested bytes, it doesn't make sense to
        // cache the block's deserialized objects since that caching may not have a payoff.
        diskStore.getBytes(blockId)
      } else if (level.useMemory && memoryStore.contains(blockId)) {
        // The block was not found on disk, so serialize an in-memory copy:
        new ByteBufferBlockData(serializerManager.dataSerializeWithExplicitClassTag(
          blockId, memoryStore.getValues(blockId).get, info.classTag), true)
      } else {
        handleLocalReadFailure(blockId)
      }
    } else {  // storage level is serialized
      if (level.useMemory && memoryStore.contains(blockId)) {
        new ByteBufferBlockData(memoryStore.getBytes(blockId).get, false)
      } else if (level.useDisk && diskStore.contains(blockId)) {
        val diskData = diskStore.getBytes(blockId)
        maybeCacheDiskBytesInMemory(info, blockId, level, diskData)
        .map(new ByteBufferBlockData(_, false))
        .getOrElse(diskData)
      } else {
        handleLocalReadFailure(blockId)
      }
    }
  }

  def getLocalBytes(blockId: BlockId): Option[BlockData] = {
    logDebug(s"Getting local block $blockId as bytes")
    // As an optimization for map output fetches, if the block is for a shuffle, return it
    // without acquiring a lock; the disk store never deletes (recent) items so this should work
    if (blockId.isShuffle) {
      val shuffleBlockResolver = shuffleManager.shuffleBlockResolver
      // TODO: This should gracefully handle case where local block is not available. Currently
      // downstream code will throw an exception.
      val buf = new ChunkedByteBuffer(
        shuffleBlockResolver.getBlockData(blockId.asInstanceOf[ShuffleBlockId]).nioByteBuffer())
      Some(new ByteBufferBlockData(buf, true))
    } else {
      blockInfoManager.lockForReading(blockId).map { info => doGetLocalBytes(blockId, info) }
    }
  }
  ```

- `getLocalValues()`: 与`getLocalBytes()`类似，**从本地存储体系中获得封装为`BlockResult`的未序列化的block**

  - 优先从内存中获取未序列化的block，如果存储等级是序列化则反序列化block后返回
  - 其次从硬盘中获取block(序列化)。如果存储等级是序列化，则反序列化block后返回，并**尝试将block(未序列化)放入内存中**。如果存储等级是未序列化，则**尝试将block(序列化)放入内存中**，并反序列化block(序列化)后返回

- `getBlockData()`: **对`getLocalBytes()`的进一步封装，返回`ManagedBuffer`，**如果获取不到block，调用`reportBlockStatus()`报告block不存在

  ```scala
  override def getBlockData(blockId: BlockId): ManagedBuffer = {
    if (blockId.isShuffle) {
   shuffleManager.shuffleBlockResolver.getBlockData(blockId.asInstanceOf[ShuffleBlockId])
    } else {
      getLocalBytes(blockId) match {
        case Some(blockData) =>
        new BlockManagerManagedBuffer(blockInfoManager, blockId, blockData, true)
        case None =>
        // If this block manager receives a request for a block that it doesn't have then it's
        // likely that the master has outdated block statuses for this block. Therefore, we send
        // an RPC so that this block is marked as being unavailable from this block manager.
        reportBlockStatus(blockId, BlockStatus.empty)
        throw new BlockNotFoundException(blockId.toString)
      }
    }
  }
  ```

- `putBytes()`: **将block数据(已序列化)写入本地内存**

  - 先调用`doPutBytes()`，里面定义了写入block的函数体作为`putBody`代码块传入`doPut()`中并调用`doPut()`
  - 在`doPut()`方法中，首先获得block的写锁，调用`putBody`执行block的写入
  - 在`putBody`代码块中，如果复制因子大于1，则创建异步线程调用`replicate()`将block复制并传输其他节点中。
    - 优先写入内存中，如果存储等级是未序列化，这时需要先反序列化，再调用`MemoryStore.putIteratorAsValues()`迭代放入内存中防止出现OOM。如果存储等级是序列化则直接调用`MemoryStore.putBytes()`，这里不需要用迭代方式写入内存是因为不需要额外的展开block的内存
    - 如果内存不足则写入磁盘。
    - 根据`tellMaster`向`BlockManagerMaster`报告block状态并等待其他复制执行完毕
  - 如果写入成功并且需要保持读锁(当前任务写入block后需要读)，则调用`BlockInfoManager.downgradeLock()`将写锁降级为读锁，否则释放当前任务的所有锁
  - 如果写入失败或者发生异常则调用`removeBlockInternal()`移除这个block，在`MemoryStore`, `DiskStore`, `BlockInfoManager`中清除这个block

  ```scala
  private def doPut[T](
    blockId: BlockId,
    level: StorageLevel,
    classTag: ClassTag[_],
    tellMaster: Boolean,
    keepReadLock: Boolean)(putBody: BlockInfo => Option[T]): Option[T] = {

    require(blockId != null, "BlockId is null")
    require(level != null && level.isValid, "StorageLevel is null or invalid")

    val putBlockInfo = {
      val newInfo = new BlockInfo(level, classTag, tellMaster)
      if (blockInfoManager.lockNewBlockForWriting(blockId, newInfo)) {
        newInfo
      } else {
        logWarning(s"Block $blockId already exists on this machine; not re-adding it")
        if (!keepReadLock) {
          // lockNewBlockForWriting returned a read lock on the existing block, so we must free it:
          releaseLock(blockId)
        }
        return None
      }
    }

    val startTimeMs = System.currentTimeMillis
    var exceptionWasThrown: Boolean = true
    val result: Option[T] = try {
      val res = putBody(putBlockInfo)
      exceptionWasThrown = false
      if (res.isEmpty) {
        // the block was successfully stored
        if (keepReadLock) {
          blockInfoManager.downgradeLock(blockId)
        } else {
          blockInfoManager.unlock(blockId)
        }
      } else {
        removeBlockInternal(blockId, tellMaster = false)
        logWarning(s"Putting block $blockId failed")
      }
      res
    } catch {
      // Since removeBlockInternal may throw exception,
      // we should print exception first to show root cause.
      case NonFatal(e) =>
      logWarning(s"Putting block $blockId failed due to exception $e.")
      throw e
    } finally {
      // This cleanup is performed in a finally block rather than a `catch` to avoid having to
      // catch and properly re-throw InterruptedException.
      if (exceptionWasThrown) {
        // If an exception was thrown then it's possible that the code in `putBody` has already
        // notified the master about the availability of this block, so we need to send an update
        // to remove this block location.
        removeBlockInternal(blockId, tellMaster = tellMaster)
        // The `putBody` code may have also added a new block status to TaskMetrics, so we need
        // to cancel that out by overwriting it with an empty block status. We only do this if
        // the finally block was entered via an exception because doing this unconditionally would
        // cause us to send empty block statuses for every block that failed to be cached due to
        // a memory shortage (which is an expected failure, unlike an uncaught exception).
        addUpdatedBlockStatusToTaskMetrics(blockId, BlockStatus.empty)
      }
    }
    if (level.replication > 1) {
      logDebug("Putting block %s with replication took %s"
               .format(blockId, Utils.getUsedTimeMs(startTimeMs)))
    } else {
      logDebug("Putting block %s without replication took %s"
               .format(blockId, Utils.getUsedTimeMs(startTimeMs)))
    }
    result
  }

  private def doPutBytes[T](
    blockId: BlockId,
    bytes: ChunkedByteBuffer,
    level: StorageLevel,
    classTag: ClassTag[T],
    tellMaster: Boolean = true,
    keepReadLock: Boolean = false): Boolean = {
    doPut(blockId, level, classTag, tellMaster = tellMaster, keepReadLock = keepReadLock) { info =>
      val startTimeMs = System.currentTimeMillis
      // Since we're storing bytes, initiate the replication before storing them locally.
      // This is faster as data is already serialized and ready to send.
      val replicationFuture = if (level.replication > 1) {
        Future {
          // This is a blocking action and should run in futureExecutionContext which is a cached
          // thread pool. The ByteBufferBlockData wrapper is not disposed of to avoid releasing
          // buffers that are owned by the caller.
          replicate(blockId, new ByteBufferBlockData(bytes, false), level, classTag)
        }(futureExecutionContext)
      } else {
        null
      }

      val size = bytes.size

      if (level.useMemory) {
        // Put it in memory first, even if it also has useDisk set to true;
        // We will drop it to disk later if the memory store can't hold it.
        val putSucceeded = if (level.deserialized) {
          val values =
          serializerManager.dataDeserializeStream(blockId, bytes.toInputStream())(classTag)
          memoryStore.putIteratorAsValues(blockId, values, classTag) match {
            case Right(_) => true
            case Left(iter) =>
            // If putting deserialized values in memory failed, we will put the bytes directly to
            // disk, so we don't need this iterator and can close it to free resources earlier.
            iter.close()
            false
          }
        } else {
          val memoryMode = level.memoryMode
          memoryStore.putBytes(blockId, size, memoryMode, () => {
            if (memoryMode == MemoryMode.OFF_HEAP &&
                bytes.chunks.exists(buffer => !buffer.isDirect)) {
              bytes.copy(Platform.allocateDirectBuffer)
            } else {
              bytes
            }
          })
        }
        if (!putSucceeded && level.useDisk) {
          logWarning(s"Persisting block $blockId to disk instead.")
          diskStore.putBytes(blockId, bytes)
        }
      } else if (level.useDisk) {
        diskStore.putBytes(blockId, bytes)
      }

      val putBlockStatus = getCurrentBlockStatus(blockId, info)
      val blockWasSuccessfullyStored = putBlockStatus.storageLevel.isValid
      if (blockWasSuccessfullyStored) {
        // Now that the block is in either the memory or disk store,
        // tell the master about it.
        info.size = size
        if (tellMaster && info.tellMaster) {
          reportBlockStatus(blockId, putBlockStatus)
        }
        addUpdatedBlockStatusToTaskMetrics(blockId, putBlockStatus)
      }
      logDebug("Put block %s locally took %s".format(blockId, Utils.getUsedTimeMs(startTimeMs)))
      if (level.replication > 1) {
        // Wait for asynchronous replication to finish
        try {
          ThreadUtils.awaitReady(replicationFuture, Duration.Inf)
        } catch {
          case NonFatal(t) =>
          throw new Exception("Error occurred while waiting for replication to finish", t)
        }
      }
      if (blockWasSuccessfullyStored) {
        None
      } else {
        Some(bytes)
      }
    }.isEmpty
  }

  def putBytes[T: ClassTag](
    blockId: BlockId,
    bytes: ChunkedByteBuffer,
    level: StorageLevel,
    tellMaster: Boolean = true): Boolean = {
    require(bytes != null, "Bytes is null")
    doPutBytes(blockId, bytes, level, implicitly[ClassTag[T]], tellMaster)
  }
  ```

- `doPutIterator()`: 与`doPutBytes()`类似，**将未序列化的数据写入block**。优先写到内存中，都是调用了迭代写入，内存不足则写到硬盘中

- `putIterator()`: 实际调用了`doPutIterator()`

- `putBlockdata()`: 实际调用了`putBytes()`

- `getMatchingBlockIds()`: 获得`blockInfoManager`和`diskBlockManager`中符合条件的block，`DiskBlockManager`中可能存在`BlockInfoManager`不知道的Block

  ```scala
  def getMatchingBlockIds(filter: BlockId => Boolean): Seq[BlockId] = {
    // The `toArray` is necessary here in order to force the list to be materialized so that we
    // don't try to serialize a lazy iterator when responding to client requests.
    (blockInfoManager.entries.map(_._1) ++ diskBlockManager.getAllBlocks())
    .filter(filter)
    .toArray
    .toSeq
  }
  ```

- `getRemoteBytes()`: **从远端的`blockManager`获取序列化后的数据**

  - 由`BlockManagerMaster.getLocationsAndStatus()`获得block位置信息，因为所有的远程block都注册在driver上
  - 解析出block大小和所有持有该block的`BlockManagerId`保存到`blockLocations`序列中
  - 调用`sortLocations()`对`blockLocations`进行排序，先进行随机排序，再**根据是否是同一机器，是否和本地同属于一个机架分成三部分，进行排序**
  - 按顺序依次从排序好的`locations`位置调用`blockTransferService.fetchBlockSync()`获取block直到获取到block

  ```scala
  private def sortLocations(locations: Seq[BlockManagerId]): Seq[BlockManagerId] = {
    val locs = Random.shuffle(locations)
    val (preferredLocs, otherLocs) = locs.partition { loc => blockManagerId.host == loc.host }
    blockManagerId.topologyInfo match {
      case None => preferredLocs ++ otherLocs
      case Some(_) =>
      val (sameRackLocs, differentRackLocs) = otherLocs.partition {
        loc => blockManagerId.topologyInfo == loc.topologyInfo
      }
      preferredLocs ++ sameRackLocs ++ differentRackLocs
    }
  }

  def getRemoteBytes(blockId: BlockId): Option[ChunkedByteBuffer] = {
    logDebug(s"Getting remote block $blockId")
    require(blockId != null, "BlockId is null")
    var runningFailureCount = 0
    var totalFailureCount = 0

    // Because all the remote blocks are registered in driver, it is not necessary to ask
    // all the slave executors to get block status.
    val locationsAndStatus = master.getLocationsAndStatus(blockId)
    val blockSize = locationsAndStatus.map { b =>
      b.status.diskSize.max(b.status.memSize)
    }.getOrElse(0L)
    val blockLocations = locationsAndStatus.map(_.locations).getOrElse(Seq.empty)

    // If the block size is above the threshold, we should pass our FileManger to
    // BlockTransferService, which will leverage it to spill the block; if not, then passed-in
    // null value means the block will be persisted in memory.
    val tempFileManager = if (blockSize > maxRemoteBlockToMem) {
      remoteBlockTempFileManager
    } else {
      null
    }

    val locations = sortLocations(blockLocations)
    val maxFetchFailures = locations.size
    var locationIterator = locations.iterator
    while (locationIterator.hasNext) {
      val loc = locationIterator.next()
      logDebug(s"Getting remote block $blockId from $loc")
      val data = try {
        blockTransferService.fetchBlockSync(
          loc.host, loc.port, loc.executorId, blockId.toString, tempFileManager)
      } catch {
        case NonFatal(e) =>
        runningFailureCount += 1
        totalFailureCount += 1

        if (totalFailureCount >= maxFetchFailures) {
          // Give up trying anymore locations. Either we've tried all of the original locations,
          // or we've refreshed the list of locations from the master, and have still
          // hit failures after trying locations from the refreshed list.
          logWarning(s"Failed to fetch block after $totalFailureCount fetch failures. " +
                     s"Most recent failure cause:", e)
          return None
        }

        logWarning(s"Failed to fetch remote block $blockId " +
                   s"from $loc (failed attempt $runningFailureCount)", e)

        // If there is a large number of executors then locations list can contain a
        // large number of stale entries causing a large number of retries that may
        // take a significant amount of time. To get rid of these stale entries
        // we refresh the block locations after a certain number of fetch failures
        if (runningFailureCount >= maxFailuresBeforeLocationRefresh) {
          locationIterator = sortLocations(master.getLocations(blockId)).iterator
          logDebug(s"Refreshed locations from the driver " +
                   s"after ${runningFailureCount} fetch failures.")
          runningFailureCount = 0
        }

        // This location failed, so we retry fetch from a different one by returning null here
        null
      }

      if (data != null) {
        // SPARK-24307 undocumented "escape-hatch" in case there are any issues in converting to
        // ChunkedByteBuffer, to go back to old code-path.  Can be removed post Spark 2.4 if
        // new path is stable.
        if (remoteReadNioBufferConversion) {
          return Some(new ChunkedByteBuffer(data.nioByteBuffer()))
        } else {
          return Some(ChunkedByteBuffer.fromManagedBuffer(data))
        }
      }
      logDebug(s"The value of block $blockId is null")
    }
    logDebug(s"Block $blockId not found")
    None
  }
  ```

- `getRemoteValues()`: 调用了`getRemoteBytes()`再进行反序列化

- `get()`: **先调用`getLocalValues()`尝试从本地获取block，如果获取不到再调用`getRemoteValues()`从远端获取block**

- `getOrElseUpdate()`: **尝试调用`get()`获取block，如果不存在则调用`makeIterator()`计算block**，调用`doPutIterator()`

  ```scala
  def getOrElseUpdate[T](
    blockId: BlockId,
    level: StorageLevel,
    classTag: ClassTag[T],
    makeIterator: () => Iterator[T]): Either[BlockResult, Iterator[T]] = {
    // Attempt to read the block from local or remote storage. If it's present, then we don't need
    // to go through the local-get-or-put path.
    get[T](blockId)(classTag) match {
      case Some(block) =>
      return Left(block)
      case _ =>
      // Need to compute the block.
    }
    // Initially we hold no locks on this block.
    doPutIterator(blockId, makeIterator, level, classTag, keepReadLock = true) match {
      case None =>
      // doPut() didn't hand work back to us, so the block already existed or was successfully
      // stored. Therefore, we now hold a read lock on the block.
      val blockResult = getLocalValues(blockId).getOrElse {
        // Since we held a read lock between the doPut() and get() calls, the block should not
        // have been evicted, so get() not returning the block indicates some internal error.
        releaseLock(blockId)
        throw new SparkException(s"get() failed for block $blockId even though we held a lock")
      }
      // We already hold a read lock on the block from the doPut() call and getLocalValues()
      // acquires the lock again, so we need to call releaseLock() here so that the net number
      // of lock acquisitions is 1 (since the caller will only call release() once).
      releaseLock(blockId)
      Left(blockResult)
      case Some(iter) =>
      // The put failed, likely because the data was too large to fit in memory and could not be
      // dropped to disk. Therefore, we need to pass the input iterator back to the caller so
      // that they can decide what to do with the values (e.g. process them without caching).
      Right(iter)
    }
  }
  ```

- `getSingle()`: 调用了`get()`方法并返回第一个对象

- `putSingle()`: 调用了`putIterator()`方法写入一个未序列化对象

- `dropFromMemory()`: 从内存中删除block，如果可行，存储等级降级为硬盘

  ```scala
  private[storage] override def dropFromMemory[T: ClassTag](
    blockId: BlockId,
    data: () => Either[Array[T], ChunkedByteBuffer]): StorageLevel = {
    logInfo(s"Dropping block $blockId from memory")
    val info = blockInfoManager.assertBlockIsLockedForWriting(blockId)
    var blockIsUpdated = false
    val level = info.level

    // Drop to disk, if storage level requires
    if (level.useDisk && !diskStore.contains(blockId)) {
      logInfo(s"Writing block $blockId to disk")
      data() match {
        case Left(elements) =>
        diskStore.put(blockId) { channel =>
          val out = Channels.newOutputStream(channel)
          serializerManager.dataSerializeStream(
            blockId,
            out,
            elements.toIterator)(info.classTag.asInstanceOf[ClassTag[T]])
        }
        case Right(bytes) =>
        diskStore.putBytes(blockId, bytes)
      }
      blockIsUpdated = true
    }

    // Actually drop from memory store
    val droppedMemorySize =
    if (memoryStore.contains(blockId)) memoryStore.getSize(blockId) else 0L
    val blockIsRemoved = memoryStore.remove(blockId)
    if (blockIsRemoved) {
      blockIsUpdated = true
    } else {
      logWarning(s"Block $blockId could not be dropped from memory as it does not exist")
    }

    val status = getCurrentBlockStatus(blockId, info)
    if (info.tellMaster) {
      reportBlockStatus(blockId, status, droppedMemorySize)
    }
    if (blockIsUpdated) {
      addUpdatedBlockStatusToTaskMetrics(blockId, status)
    }
    status.storageLevel
  }
  ```

- `removeBlock()`: 本地存储体系中删除一个block。实际调用了`removeBlockInternal()`，分别调用了`MemoryStore.remove()`和`DiskStore.remove()`从内存和硬盘移除block，再调用`BlockInfoManager.removeBlock()`移除block的元信息
- `removeRdd()`: 调用`removeBlock()`移除某个RDD的所有block
- `removeBroadcast()`: 调用`removeBlock()`移除某个广播变量的所有block

## `BlockManagerMaster`

**对存在于Executor或Driver上的`BlockManager`进行统一管理**，Executor与Driver关于`BlockManager`的交互都依赖于`BlockManagerMaster`。

`BlockManagerMaster`通过[前文所述的RPC组件]({% post_url 2020-9-3-RpcEnv %})进行通信。Driver上的`BlockManagerMaster`持有`BlockManagerMasterEndpoint`，所有的`BlockManagerMaster`持有自己的`BlockManagerSlaveEndpoint`。所有的`BlockManagerMaster`的`driverEndpoint`属性都持有`BlockManagerMasterEndpoint`的`RpcEndpointRef`。所有的`BlockManagerMaster`的`slaveEndpoint`属性都持有`BlockManagerSlaveEndpoint`的`RpcEndpointRef`

## REFERENCE

1. Spark内核设计的艺术：架构设计与实现
