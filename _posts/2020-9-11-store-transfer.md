---
layout: post
title:  "Spark源码阅读(十一)：存储体系之block传输"
date:   2020-9-11
categories: Spark
keywords: Spark, 存储体系
mathjax: false
author: wzx
---

介绍Spark中的block传输





## `DiskBlockObjectWriter`

将在shuffle阶段将map task的输出写入磁盘，这样reduce task就能够从磁盘中获取map task的中间输出了。有以下重要成员属性

- `file`: 要写入的文件

- `bufferSize`: 缓冲大小
- `syncWrites`: 是否同步写
- `blockId`: `BlockId`
- `channel`: `FileChannel`
- `mcs`: `ManualCloseOutputStream`
- `bs`: `OutputStream`
- `fos`: `FileOutputStream`
- `objOut`: `SerializationStream`
- `ts`: `TimeTrackingOutputStream`
- `initialized`: 是否已经初始化
- `streamOpen`: 是否已经打开流
- `hasBeenClosed`: 是否已经关闭
- `committedPosition`: 提交的文件位置
- `reportedPosition`: 报告给度量系统的文件位置
- `numRecordsWritten`: 已写的记录数

下面是重要的方法

- `open()`: 打开要写入文件的各种输出流及管道。由此初始化方法可以看出来，`blockId`只有在创建`bs`(`OutputStream`)时使用到了，并且可以看出**`blockId`只在是否压缩时使用到了，文件输出流由`file`成员属性决定，即最终写入的文件由`file`决定**。

  ```scala
  private def initialize(): Unit = {
    fos = new FileOutputStream(file, true)
    channel = fos.getChannel()
    ts = new TimeTrackingOutputStream(writeMetrics, fos)
    class ManualCloseBufferedOutputStream
    extends BufferedOutputStream(ts, bufferSize) with ManualCloseOutputStream
    mcs = new ManualCloseBufferedOutputStream
  }

  def open(): DiskBlockObjectWriter = {
    if (hasBeenClosed) {
      throw new IllegalStateException("Writer already closed. Cannot be reopened.")
    }
    if (!initialized) {
      initialize()
      initialized = true
    }

    bs = serializerManager.wrapStream(blockId, mcs)
    objOut = serializerInstance.serializeStream(bs)
    streamOpen = true
    this
  }

  // SerializerManager.scala
  def wrapStream(blockId: BlockId, s: OutputStream): OutputStream = {
    wrapForCompression(blockId, wrapForEncryption(s))
  }
  def wrapForCompression(blockId: BlockId, s: OutputStream): OutputStream = {
    if (shouldCompress(blockId)) compressionCodec.compressedOutputStream(s) else s
  }
  private def shouldCompress(blockId: BlockId): Boolean = {
    blockId match {
      case _: ShuffleBlockId => compressShuffle
      case _: BroadcastBlockId => compressBroadcast
      case _: RDDBlockId => compressRdds
      case _: TempLocalBlockId => compressShuffleSpill
      case _: TempShuffleBlockId => compressShuffle
      case _ => false
    }
  }
  ```

- `recordWritten()`: 用于对写入的记录数进行统计

  ```scala
  private def updateBytesWritten() {
    val pos = channel.position()
    writeMetrics.incBytesWritten(pos - reportedPosition)
    reportedPosition = pos
  }

  def recordWritten(): Unit = {
    numRecordsWritten += 1
    writeMetrics.incRecordsWritten(1)

    // 2^14
    if (numRecordsWritten % 16384 == 0) {
      updateBytesWritten()
    }
  }
  ```

- `write()`: 向输出流中写入数据

  ```scala
  override def write(kvBytes: Array[Byte], offs: Int, len: Int): Unit = {
    if (!streamOpen) {
      open()
    }

    bs.write(kvBytes, offs, len)
  }
  ```

- `commitAndGet()`: 把输出流中的数据写入文件并返回`FileSegment`。`syncWrites`表示强制等待当前流写完

  ```scala
  def commitAndGet(): FileSegment = {
    if (streamOpen) {
      // NOTE: Because Kryo doesn't flush the underlying stream we explicitly flush both the
      //       serializer stream and the lower level stream.
      objOut.flush()
      bs.flush()
      objOut.close()
      streamOpen = false

      if (syncWrites) {
        // Force outstanding writes to disk and track how long it takes
        val start = System.nanoTime()
        fos.getFD.sync()
        writeMetrics.incWriteTime(System.nanoTime() - start)
      }

      val pos = channel.position()
      val fileSegment = new FileSegment(file, committedPosition, pos - committedPosition)
      committedPosition = pos
      // In certain compression codecs, more bytes are written after streams are closed
      writeMetrics.incBytesWritten(committedPosition - reportedPosition)
      reportedPosition = committedPosition
      numRecordsWritten = 0
      fileSegment
    } else {
      new FileSegment(file, committedPosition, 0)
    }
  }
  ```

## `BlockManagerMessages`

**由`BlockManagerMaster`负责发送和接收的消息**

- `RemoveExecutor`: 移除Executor
- `RegisterBlockManager`: 注册`BlockManager`
- `UpdateBlockInfo`: 更新block信息
- `GetLocations`: 获取block的位置
- `GetLocationsMultipleBlockIds`: 获取多个block的位置
- `GetPeers`: 获取其他`BlockManager`的`BlockManagerId`
- `GetExecutorEndpointRef`: 获取Executor的`EndpointRef`引用
- `RemoveBlock`: 移除block
- `RemoveRdd`: 移除RDD block
- `RemoveShuffle`: 移除Shuffle block
- `RemoveBroadcast`: 移除Broadcast block
- `GetMemoryStatus`: 获取指定的`BlockManager`的内存状态
- `GetStorageStatus`: 获取存储状态
- `GetBlockStatus`: 获取block的状态
- `GetMatchingBlockIds`: 获取匹配过滤条件的block
- `HasCachedBlocks`: 指定的Executor上是否有缓存的block
- `StopBlockManagerMaster`: 停止`BlockManagerMaster`

## `BlockManagerMasterEndpoint`

继承了[前文所述的RPC组件]({% post_url 2020-9-3-RpcEnv %})中的`ThreadSafeRpcEndpoint`。有以下重要的成员属性

- `blockManagerInfo`: `BlockManagerId`与`BlockManagerInfo`之间映射关系
- `blockManagerIdByExecutor`: Executor ID和`BlockManagerInfo`之间映射关系
- `blockLocations`: `BlockId`和`BlockManagerId`的`set`的映射关系，记录了block的存储位置
- `topologyMapper`: 集群所有结点的拓扑结构映射

重写了特质`RpcEndpoint`的`receiveAndReply()`方法，使用模式匹配将`BlockManagerMessages`中的消息交由对应的方法去处理。

```scala
override def receiveAndReply(context: RpcCallContext): PartialFunction[Any, Unit] = {
  case RegisterBlockManager(blockManagerId, maxOnHeapMemSize, maxOffHeapMemSize, slaveEndpoint) =>
  context.reply(register(blockManagerId, maxOnHeapMemSize, maxOffHeapMemSize, slaveEndpoint))

  case _updateBlockInfo @
  UpdateBlockInfo(blockManagerId, blockId, storageLevel, deserializedSize, size) =>
  context.reply(updateBlockInfo(blockManagerId, blockId, storageLevel, deserializedSize, size))
  listenerBus.post(SparkListenerBlockUpdated(BlockUpdatedInfo(_updateBlockInfo)))

  case GetLocations(blockId) =>
  context.reply(getLocations(blockId))

  ......

}
```

## `BlockManagerSlaveEndpoint`

与`BlockManagerMasterEndpoint`的继承关系相同，同样重写了特质`RpcEndpoint`的`receiveAndReply()`方法，使用模式匹配将`BlockManagerMessages`中的消息交由对应的方法去处理。

## `OneForOneBlockFetcher`

每个chunk代表一个block，**用于`BlockTransferService`调用以完成block的下载**。下面是重要的成员变量

- `client`: `TransportClient`
- `openMessage`: `OpenBlocks`。保存远端节点的appId、execId(Executor标识)和blockIds
- `blockIds`: 字符串数组。与`openMessage.blockIds`一致
- `listener`：`BlockFetchingListener`，将在获取Block成功或失败时被回调
- `chunkCallback`：获取block成功或失败时回调，配合`BlockFetchingListener`使用

下面是主要的`start()`方法，用于开启获取流程

- 由`client`向server端发送`openMessage`消息，回调方法放入缓存client端的缓存中，具体实现[前文已经提过]({% post_url 2020-9-2-RpcTransport %}#TransportClient)
- 在回调方法中，如果回调`onSuccess()`则调用`client.fetchChunk()`获取所有block

```scala
public void start() {
  if (blockIds.length == 0) {
    throw new IllegalArgumentException("Zero-sized blockIds array");
  }

  client.sendRpc(openMessage.toByteBuffer(), new RpcResponseCallback() {
    @Override
    public void onSuccess(ByteBuffer response) {
      try {
        streamHandle = (StreamHandle) BlockTransferMessage.Decoder.fromByteBuffer(response);
        logger.trace("Successfully opened blocks {}, preparing to fetch chunks.", streamHandle);

        // Immediately request all chunks -- we expect that the total size of the request is
        // reasonable due to higher level chunking in [[ShuffleBlockFetcherIterator]].
        for (int i = 0; i < streamHandle.numChunks; i++) {
          if (downloadFileManager != null) {
            client.stream(OneForOneStreamManager.genStreamChunkId(streamHandle.streamId, i),
                          new DownloadCallback(i));
          } else {
            client.fetchChunk(streamHandle.streamId, i, chunkCallback);
          }
        }
      } catch (Exception e) {
        logger.error("Failed while starting block fetches after success", e);
        failRemainingBlocks(blockIds, e);
      }
    }

    @Override
    public void onFailure(Throwable e) {
      logger.error("Failed while starting block fetches", e);
      failRemainingBlocks(blockIds, e);
    }
  });
}

private void failRemainingBlocks(String[] failedBlockIds, Throwable e) {
  for (String blockId : failedBlockIds) {
    try {
      listener.onBlockFetchFailure(blockId, e);
    } catch (Exception e2) {
      logger.error("Error in block fetch failure callback", e2);
    }
  }
}
```

## `NettyBlockRpcServer`

继承了`RpcHandler`，用于**处理 `NettyBlockTransferService`中的 `TransportRequestHandler`中的请求消息**，作为shuffle的server端

- `streamManager`: `OneForOneStreamManager`，在[之前RPC中已经介绍过了]({% post_url 2020-9-1-RpcStreaming %}#OneForOneStreamManager)

重写了`receive()`和`putBlockDataAsStream()`

`receive()`: **主要接收并处理`OpenBlocks`和`UploadBlock`消息**

- 当接收到`OpenBlocks`消息时，获取到请求的`BlockId`，由`BlockManager.getBlockData`获取封装为`ManagedBuffer`的block数据(序列化)序列
- 调用`OneForOneStreamManager.registerStream()`将block数据序列和app id 和channel一起注册到`streams`中，`streamId`自增生成
- 创建`StreamHandle`消息（包含`streamId`和`ManagedBuffer`序列的大小），并通过响应上下文回复客户端
- 当接收到`UploadBlock`消息时，获取到请求的元数据解析得到存储等级，类型信息和`BlockId`
- 调用`BlockManager.putBlockData()`写入本地存储体系中
- 通过响应上下文回复客户端

```scala
override def receive(
  client: TransportClient,
  rpcMessage: ByteBuffer,
  responseContext: RpcResponseCallback): Unit = {
  val message = BlockTransferMessage.Decoder.fromByteBuffer(rpcMessage)
  logTrace(s"Received request: $message")

  message match {
    case openBlocks: OpenBlocks =>
    val blocksNum = openBlocks.blockIds.length
    val blocks = for (i <- (0 until blocksNum).view)
    yield blockManager.getBlockData(BlockId.apply(openBlocks.blockIds(i)))
    val streamId = streamManager.registerStream(appId, blocks.iterator.asJava,
                                                client.getChannel)
    logTrace(s"Registered streamId $streamId with $blocksNum buffers")
    responseContext.onSuccess(new StreamHandle(streamId, blocksNum).toByteBuffer)

    case uploadBlock: UploadBlock =>
    // StorageLevel and ClassTag are serialized as bytes using our JavaSerializer.
    val (level: StorageLevel, classTag: ClassTag[_]) = {
      serializer
      .newInstance()
      .deserialize(ByteBuffer.wrap(uploadBlock.metadata))
      .asInstanceOf[(StorageLevel, ClassTag[_])]
    }
    val data = new NioManagedBuffer(ByteBuffer.wrap(uploadBlock.blockData))
    val blockId = BlockId(uploadBlock.blockId)
    logDebug(s"Receiving replicated block $blockId with level ${level} " +
             s"from ${client.getSocketAddress}")
    blockManager.putBlockData(blockId, data, level, classTag)
    responseContext.onSuccess(ByteBuffer.allocate(0))
  }
}
```

## `BlockTransferService`

**用于发起block的上传和下载请求**，作为shuffle的client端。下面是具体的两个方法

- `fetchBlockSync()`: **同步下载block**，实质就是调用未实现的`fetchBlocks()`，并且等待`BlockFetchingListener`处理获取后的block数据，这里要等待两个流程(请求block，拉取block数据，具体看总结部分)

  ```scala
  def fetchBlockSync(
    host: String,
    port: Int,
    execId: String,
    blockId: String,
    tempFileManager: DownloadFileManager): ManagedBuffer = {
    // A monitor for the thread to wait on.
    val result = Promise[ManagedBuffer]()
    fetchBlocks(host, port, execId, Array(blockId),
                new BlockFetchingListener {
                  override def onBlockFetchFailure(blockId: String, exception: Throwable): Unit = {
                    result.failure(exception)
                  }
                  override def onBlockFetchSuccess(blockId: String, data: ManagedBuffer): Unit = {
                    data match {
                      case f: FileSegmentManagedBuffer =>
                      result.success(f)
                      case e: EncryptedManagedBuffer =>
                      result.success(e)
                      case _ =>
                      try {
                        val ret = ByteBuffer.allocate(data.size.toInt)
                        ret.put(data.nioByteBuffer())
                        ret.flip()
                        result.success(new NioManagedBuffer(ret))
                      } catch {
                        case e: Throwable => result.failure(e)
                      }
                    }
                  }
                }, tempFileManager)
    ThreadUtils.awaitResult(result.future, Duration.Inf)
  }
  ```

- `uploadBlockSync()`: **同步上传block**，内部调用了未实现的`uploadBlock()`方法，同步的实现方式与`fetchBlockSync()`相同，只不过`Promise`对象在`uploadBlock()`方法内部创建了

  ```scala
  def uploadBlockSync(
    hostname: String,
    port: Int,
    execId: String,
    blockId: BlockId,
    blockData: ManagedBuffer,
    level: StorageLevel,
    classTag: ClassTag[_]): Unit = {
    val future = uploadBlock(hostname, port, execId, blockId, blockData, level, classTag)
    ThreadUtils.awaitResult(future, Duration.Inf)
  }
  ```



`BlockTransferService`有两个实现类：用于测试的`MockBlockTransferService`及`NettyBlockTransferService`

下面主要介绍`NettyBlockTransferService`。有以下重要成员对象

- `transportContext`: `TransportContext`
- `server`: `TransportServer`

下面是重要的成员方法

- `init()`: `NettyBlockTransferService`只有在其`init()`方法被调用，即被初始化后才提供服务

  ```scala
  private def createServer(bootstraps: List[TransportServerBootstrap]): TransportServer = {
    def startService(port: Int): (TransportServer, Int) = {
      val server = transportContext.createServer(bindAddress, port, bootstraps.asJava)
      (server, server.getPort)
    }

    Utils.startServiceOnPort(_port, startService, conf, getClass.getName)._1
  }

  override def init(blockDataManager: BlockDataManager): Unit = {
    val rpcHandler = new NettyBlockRpcServer(conf.getAppId, serializer, blockDataManager)
    var serverBootstrap: Option[TransportServerBootstrap] = None
    var clientBootstrap: Option[TransportClientBootstrap] = None
    if (authEnabled) {
      serverBootstrap = Some(new AuthServerBootstrap(transportConf, securityManager))
      clientBootstrap = Some(new AuthClientBootstrap(transportConf, conf.getAppId, securityManager))
    }
    transportContext = new TransportContext(transportConf, rpcHandler)
    clientFactory = transportContext.createClientFactory(clientBootstrap.toSeq.asJava)
    server = createServer(serverBootstrap.toList)
    appId = conf.getAppId
    logInfo(s"Server created on ${hostName}:${server.getPort}")
  }
  ```

- `fetchBlocks()`: **发送下载远端block请求**

  - 创建`RetryingBlockFetcher.BlockFetchStarter`，如果最大重试次数大于0则创建`RetryingBlockFetcher`用于发送请求，否则直接调用`blockFetchStarter`的`createAndStart()`方法，创建`TransportClient`再创建`OneForOneBlockFetcher`并调用`start()`

    ```scala
    override def fetchBlocks(
      host: String,
      port: Int,
      execId: String,
      blockIds: Array[String],
      listener: BlockFetchingListener,
      tempFileManager: DownloadFileManager): Unit = {
      logTrace(s"Fetch blocks from $host:$port (executor id $execId)")
      try {
        val blockFetchStarter = new RetryingBlockFetcher.BlockFetchStarter {
          override def createAndStart(blockIds: Array[String], listener: BlockFetchingListener) {
            val client = clientFactory.createClient(host, port)
            new OneForOneBlockFetcher(client, appId, execId, blockIds, listener,
                                      transportConf, tempFileManager).start()
          }
        }

        val maxRetries = transportConf.maxIORetries()
        if (maxRetries > 0) {
          // Note this Fetcher will correctly handle maxRetries == 0; we avoid it just in case there's
          // a bug in this code. We should remove the if statement once we're sure of the stability.
          new RetryingBlockFetcher(transportConf, blockFetchStarter, blockIds, listener).start()
        } else {
          blockFetchStarter.createAndStart(blockIds, listener)
        }
      } catch {
        case e: Exception =>
        logError("Exception while beginning fetchBlocks", e)
        blockIds.foreach(listener.onBlockFetchFailure(_, e))
      }
    }
    ```

  - 在`RetryingBlockFetcher`中实际上调用了`fetchAllOutstanding()`方法。内部发送获取block请求部分其实是调用的`fetchBlocks()`中传入的`blockFetchStarter`的`createAndStart()`方法与非重试的fetcher请求代码是一样的

    ```scala
    // RetryingBlockFetcher
    public void start() {
      fetchAllOutstanding();
    }

    private void fetchAllOutstanding() {
      // Start by retrieving our shared state within a synchronized block.
      String[] blockIdsToFetch;
      int numRetries;
      RetryingBlockFetchListener myListener;
      synchronized (this) {
        blockIdsToFetch = outstandingBlocksIds.toArray(new String[outstandingBlocksIds.size()]);
        numRetries = retryCount;
        myListener = currentListener;
      }

      // Now initiate the fetch on all outstanding blocks, possibly initiating a retry if that fails.
      try {
        fetchStarter.createAndStart(blockIdsToFetch, myListener);
      } catch (Exception e) {
        logger.error(String.format("Exception while beginning fetch of %s outstanding blocks %s",
                                   blockIdsToFetch.length, numRetries > 0 ? "(after " + numRetries + " retries)" : ""), e);

        if (shouldRetry(e)) {
          initiateRetry();
        } else {
          for (String bid : blockIdsToFetch) {
            listener.onBlockFetchFailure(bid, e);
          }
        }
      }
    }
    ```

  - 如果出现异常会调用`shouldRetry()`判断是否需要重试，异常是`IOException`并且当前的重试次数`retryCount`小于最大重试次数`maxRetries`。之后调用`initiateRetry()`方法进行重试，重试次数+1，新建`RetryingBlockFetchListener`后，使用线程池(`Executors.newCachedThreadPool`)提交重试任务(等待一段时间后，再次调用`fetchAllOutstanding()`)，**重试机制是异步的**

    ```scala
    // RetryingBlockFetcher
    private synchronized boolean shouldRetry(Throwable e) {
      boolean isIOException = e instanceof IOException
      || (e.getCause() != null && e.getCause() instanceof IOException);
      boolean hasRemainingRetries = retryCount < maxRetries;
      return isIOException && hasRemainingRetries;
    }

    private synchronized void initiateRetry() {
      retryCount += 1;
      currentListener = new RetryingBlockFetchListener();

      logger.info("Retrying fetch ({}/{}) for {} outstanding blocks after {} ms",
                  retryCount, maxRetries, outstandingBlocksIds.size(), retryWaitTime);

      executorService.submit(() -> {
        Uninterruptibles.sleepUninterruptibly(retryWaitTime, TimeUnit.MILLISECONDS);
        fetchAllOutstanding();
      });
    }
    ```

- `uploadBlock()`: **发送向远端上传block的请求**

  - 创建`TransportClient`
  - 序列化存储等级和类型
  - 如果大于`spark.maxRemoteBlockSizeFetchToMem`则调动`TransportClient.uploadStream()`作为流将block上传
  - 否则调用`TransportClient.sendRpc()`直接将block数据作为字节数组上传

  ```scala
  override def uploadBlock(
    hostname: String,
    port: Int,
    execId: String,
    blockId: BlockId,
    blockData: ManagedBuffer,
    level: StorageLevel,
    classTag: ClassTag[_]): Future[Unit] = {
    val result = Promise[Unit]()
    val client = clientFactory.createClient(hostname, port)

    // StorageLevel and ClassTag are serialized as bytes using our JavaSerializer.
    // Everything else is encoded using our binary protocol.
    val metadata = JavaUtils.bufferToArray(serializer.newInstance().serialize((level, classTag)))

    val asStream = blockData.size() > conf.get(config.MAX_REMOTE_BLOCK_SIZE_FETCH_TO_MEM)
    val callback = new RpcResponseCallback {
      override def onSuccess(response: ByteBuffer): Unit = {
        logTrace(s"Successfully uploaded block $blockId${if (asStream) " as stream" else ""}")
        result.success((): Unit)
      }

      override def onFailure(e: Throwable): Unit = {
        logError(s"Error while uploading $blockId${if (asStream) " as stream" else ""}", e)
        result.failure(e)
      }
    }
    if (asStream) {
      val streamHeader = new UploadBlockStream(blockId.name, metadata).toByteBuffer
      client.uploadStream(new NioManagedBuffer(streamHeader), blockData, callback)
    } else {
      // Convert or copy nio buffer into array in order to serialize it.
      val array = JavaUtils.bufferToArray(blockData.nioByteBuffer())

      client.sendRpc(new UploadBlock(appId, execId, blockId.name, metadata, array).toByteBuffer,
                     callback)
    }

    result.future
  }
  ```



## 总结

异步获取远端block的流程如下

<img src="{{ site.url }}/assets/img/2020-9-11-1.png" style="zoom: 67%;" />

1. client端调用`NettyBlockTransferService.fetchBlocks()`方法获取block，如果指定重试次数，则调用`RetryingBlockFetcher.start()`，内部调用`BlockFetchStarter.createAndStart()`并且失败异步重试，同时传入`BlockFetchingListener`用于获取block到本地后的成功或者失败回调函数(观察者模式)
2. 如果未指定重试次数，则是直接调用`BlockFetchStarter.createAndStart()`，失败不重试
3. `BlockFetchStarter.createAndStart()`将调用`OneForOneBlockFetcher.start()`
4. `OneForOneBlockFetcher.start()`将调用`TransportClient.sendRpc()`方法发送`OpenBlocks`消息。接着用于处理server端响应的回调函数在`TransportResponseHandler`中注册
5. server端的`TransportRequestHandler`接收到了client端发送的消息，然后调用 `NettyBlockRpcServer`处理`OpenBlocks`消息
6. `NettyBlockRpcServer.receive()`中循环调用`BlockManager.getBlockData()`在本地存储系统中获取block数据序列
7. 接着调用`OneForOneStreamManager.registerStream()`，将block数据序列注册为一个流
8. 创建`StreamHandle`包含流Id和block数据序列大小通过成功回调函数响应给client端
9. client端的`TransportResponseHandler`接收到server端的响应消息，然后调用之前注册的回调函数，迭代请求返回的流id中的所有chunk id(chunk与block意义相同)，调用`TransportClient.fetchChunk()`
10. 这个方法调用后与1~9的流程类似，先想server端请求，server端再将block数据返回具体如下。从远端节点的缓存streams中找到与streamId对应的`StreamState`，并根据索引返回`StreamState`的`ManagedBuffer`序列中的某一个`ManagedBuffer`(block数据)。最后调用步骤1和2中注册的观察者`BlockFetchingListener`处理获取到block数据

异步上传block至远端的流程如下

<img src="{{ site.url }}/assets/img/2020-9-11-2.png" style="zoom:50%;" />

1. client端要通过`BlockManager`获取block数据，转化为`ManagedBuffer`类型
2. 调用`NettyBlockTransferService.uploadBlock()`方法上传block到远端节点
3. 内部调用了`TransportClient.sendRpc()`方法发送`UploadBlock`消息，接着用于处理server端响应的回调函数在`TransportResponseHandler`中注册
4. server端的`TransportRequestHandler`接收到了client端发送的消息，然后调用 `NettyBlockRpcServer`处理`UploadBlock`消息。`NettyBlockRpcServer.receive()`中循环调用`BlockManager.putBlockData()`将block数据写入本地存储系统中
5. `NettyBlockRpcServer`将处理成功的消息返回给客户端，client端的`TransportResponseHandler`接收到server端的响应消息，然后调用之前注册的回调函数

## REFERENCE

1. Spark内核设计的艺术：架构设计与实现
