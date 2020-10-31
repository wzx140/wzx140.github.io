---
layout: post
title:  "Spark源码阅读(三)：RPC之StreamManager、RpcHandler"
date:   2020-9-1
categories: Spark
tags: Spark SparkCore RPC
mathjax: false
author: wzx
---

- 目录
{:toc}


Spark RPC中的底层的流数据处理与消息传输




## `StreamManager`

**在流中获取单个块**，这在`TransportRequestHandler`中用于响应`fetchChunk()`请求。有两个子类`OneForOneStreamManager`和`NettyStreamManager`

### `OneForOneStreamManager`

**为`NettyBlockRpcServer`提供了一对一的流服务**。`ManagedBuffer`是一个不可变的byte数组的抽象。

内部类`StreamState`维护了单个流的状态，如下代码所示

- `appId`：请求流的应用程序id
- `buffers`：可迭代的`ManagedBuffer`，表示当前流的缓冲数据
- `associatedChannel`: 与当前流关联的channel
- `chunksBeingTransferred`: 正在传输的`ManagedBuffer`数量
- `curChunk`: 客户端当前接收到的`ManagedBuffer`索引，为了确认调用方按顺序且一次只请求一个chunk

```scala
private static class StreamState {
  final String appId;
  final Iterator<ManagedBuffer> buffers;
  final Channel associatedChannel;
  int curChunk = 0;
  volatile long chunksBeingTransferred = 0L;

  StreamState(String appId, Iterator<ManagedBuffer> buffers, Channel channel) {
    this.appId = appId;
    this.buffers = Preconditions.checkNotNull(buffers);
    this.associatedChannel = channel;
  }
}
```

`OneForOneStreamManager`有以下重要的成员属性

- `nextStreamId`：下一个stream的id，`AtomicLong`保证了并发安全。
- `streams`: 维护了stream id和`StreamState`之间的映射关系。`ConcurrentHashMap<Long, StreamState>`保证了线程安全。

以下为重要的方法

- `registerStream()`: 注册一个`ManagedBuffers`流和channel。  

  ```scala
  public long registerStream(String appId, Iterator<ManagedBuffer> buffers, Channel channel) {
    long myStreamId = nextStreamId.getAndIncrement();
    streams.put(myStreamId, new StreamState(appId, buffers, channel));
    return myStreamId;
  }
  ```

- `getChunk()`: 获取被封装为`ManagedBuffer`的单独块。如果当前流已经到达末尾，就移除这个流。

  ```scala
  public ManagedBuffer getChunk(long streamId, int chunkIndex) {
    StreamState state = streams.get(streamId);
    if (chunkIndex != state.curChunk) {
      throw new IllegalStateException(String.format(
        "Received out-of-order chunk index %s (expected %s)", chunkIndex, state.curChunk));
    } else if (!state.buffers.hasNext()) {
      throw new IllegalStateException(String.format(
        "Requested chunk index beyond end %s", chunkIndex));
    }
    state.curChunk += 1;
    ManagedBuffer nextChunk = state.buffers.next();
  
    if (!state.buffers.hasNext()) {
      logger.trace("Removing stream id {}", streamId);
      streams.remove(streamId);
    }
  
    return nextChunk;
  }
  ```

  

### `NettyStreamManager`

**为`NettyRpcEnv`提供文件流服务**。提供对普通文件，jar包和目录的下载和添加缓存的功能。`TransportRequestHandler`的`StreamRequest`消息的处理依赖于`NettyStreamManager`，各个Executor节点就可以使用Driver节点的RpcEnv提供的``NettyStreamManager`，从Driver将Jar包或文件下载到Executor节点上供任务执行。

## `RpcHandler`

**处理`TransportRequestHandler`中的请求消息**。下面主要看其实现类`NettyRpcHandler`

- `internalReceive()`: **将`ByteBuffer`封装为`RequestMessage`类型**。由`TransportClient`获取远端地址，在构造`RequestMessage`时对`ByteBuffer`进行了反序列化，若没有发送者的地址，则使用之前`TransprtClient`获取到的地址。若有发送者的地址，则在`Inbox`中添加`RemoteProcessConnected`消息

  ```scala
  private def internalReceive(client: TransportClient, message: ByteBuffer): RequestMessage = {
    val addr = client.getChannel().remoteAddress().asInstanceOf[InetSocketAddress]
    assert(addr != null)
    val clientAddr = RpcAddress(addr.getHostString, addr.getPort)
    val requestMessage = RequestMessage(nettyEnv, client, message)
    if (requestMessage.senderAddress == null) {
      // Create a new message with the socket address of the client as the sender.
      new RequestMessage(clientAddr, requestMessage.receiver, requestMessage.content)
    } else {
      // The remote RpcEnv listens to some port, we should also fire a RemoteProcessConnected for
      // the listening address
      val remoteEnvAddress = requestMessage.senderAddress
      if (remoteAddresses.putIfAbsent(clientAddr, remoteEnvAddress) == null) {
        dispatcher.postToAll(RemoteProcessConnected(remoteEnvAddress))
      }
      requestMessage
    }
  }
  
  private[netty] object RequestMessage {
  
    private def readRpcAddress(in: DataInputStream): RpcAddress = {
      val hasRpcAddress = in.readBoolean()
      if (hasRpcAddress) {
        RpcAddress(in.readUTF(), in.readInt())
      } else {
        null
      }
    }
  
    def apply(nettyEnv: NettyRpcEnv, client: TransportClient, bytes: ByteBuffer): RequestMessage = {
      val bis = new ByteBufferInputStream(bytes)
      val in = new DataInputStream(bis)
      try {
        val senderAddress = readRpcAddress(in)
        val endpointAddress = RpcEndpointAddress(readRpcAddress(in), in.readUTF())
        val ref = new NettyRpcEndpointRef(nettyEnv.conf, endpointAddress, nettyEnv)
        ref.client = client
        new RequestMessage(
          senderAddress,
          ref,
          // The remaining bytes in `bytes` are the message content.
          nettyEnv.deserialize(client, bytes))
      } finally {
        in.close()
      }
    }
  }
  
  // NettyRpcEnv.deserialize()
  private[netty] def deserialize[T: ClassTag](client: TransportClient, bytes: ByteBuffer): T = {
    NettyRpcEnv.currentClient.withValue(client) {
      deserialize { () =>
        javaSerializerInstance.deserialize[T](bytes)
      }
    }
  }
  ```

- `receive()`:  **处理一条`TransportClient `发送的 RPC 消息**。底层是将消息交[由`Dispatcher`去处理]({% post_url 2020-7-23-RpcMessage %}#dispatcher)，将消息放入`Inbox`里。

  ```scala
  // RpcEndpoint.receiveAndReply()
  override def receive(
    client: TransportClient,
    message: ByteBuffer,
    callback: RpcResponseCallback): Unit = {
    val messageToDispatch = internalReceive(client, message)
    dispatcher.postRemoteMessage(messageToDispatch, callback)
  }
  
  // RpcEndpoint.receive()
  override def receive(
    client: TransportClient,
    message: ByteBuffer): Unit = {
    val messageToDispatch = internalReceive(client, message)
    dispatcher.postOneWayMessage(messageToDispatch)
  }
  ```

- `getStreamManager()`: 获取`StreamManager`，由上一小节所述可以获取单个块

- `channelActive`：向`Inbox`投递`RemoteProcessConnected`消 息

- `channelInactive`：向`Inbox`投递`RemoteProcessDisconnected`消 息

- `exceptionCaught()`: 向`Inbox`投递`RemoteProcessConnectionError`消 息

## REFERENCE

1. [spark 源码分析](https://www.cnblogs.com/johnny666888/p/11259944.html)
2. Spark内核设计的艺术：架构设计与实现