---
layout: post
title:  "Spark源码阅读(四)：RPC之Transport传输层"
date:   2020-9-2
categories: Spark
keywords: Spark, RPC
mathjax: false
author: wzx
---

Spark Rpc中的传输层，介绍`TransportContext`, `TransportClientFactory`, `TransportResponseHandler`, `TransportRequestHandler`, `TransportChannelHandler`, `TransportClient`, `TransportServer`





## `TransportContext`

**传输上下文，内部包含传输配置信息`TransportConf`，以及对收到的RPC消息进行处理的`RpcHandler`。**

通过`createClientFactory()`方法来获得`TransportClientFactory`。通过`TransportContext`的`createServer()`方法创建传输服务端`TransportServer`的实例

```java
public TransportClientFactory createClientFactory(List<TransportClientBootstrap> bootstraps) {
  return new TransportClientFactory(this, bootstraps);
}

public TransportServer createServer(int port, List<TransportServerBootstrap> bootstraps) {
  return new TransportServer(this, null, port, rpcHandler, bootstraps);
}
```

`initializePipeline()`方法调用Netty的API对管道初始化

- 初始化了`TransportClient`，`TransportResponseHandler`，`TransportRequestHandler`，用于构造`TransportChannelHandler`
- 对管道进行设置

```java
private TransportChannelHandler createChannelHandler(Channel channel, RpcHandler rpcHandler) {
  TransportResponseHandler responseHandler = new TransportResponseHandler(channel);
  TransportClient client = new TransportClient(channel, responseHandler);
  TransportRequestHandler requestHandler = new TransportRequestHandler(channel, client,
                                                                       rpcHandler, conf.maxChunksBeingTransferred());
  return new TransportChannelHandler(client, responseHandler, requestHandler,
                                     conf.connectionTimeoutMs(), closeIdleConnections);
}

public TransportChannelHandler initializePipeline(
  SocketChannel channel,
  RpcHandler channelRpcHandler) {
  try {
    TransportChannelHandler channelHandler = createChannelHandler(channel, channelRpcHandler);
    channel.pipeline()
      .addLast("encoder", ENCODER)
      .addLast(TransportFrameDecoder.HANDLER_NAME, NettyUtils.createFrameDecoder())
      .addLast("decoder", DECODER)
      .addLast("idleStateHandler", new IdleStateHandler(0, 0, conf.connectionTimeoutMs() / 1000))
      // NOTE: Chunks are currently guaranteed to be returned in the order of request, but this
      // would require more logic to guarantee if this were not part of the same event loop.
      .addLast("handler", channelHandler);
    return channelHandler;
  } catch (RuntimeException e) {
    logger.error("Error while initializing Netty pipeline", e);
    throw e;
  }
}
```

## `Bootstrap`

`TransportClientBootstrap`, `TransportServerBootstrap` 客户端和服务端的引导程序，**在客户端和服务端初始化时执行一次，主要进行初始化的准备(验证，加密等)，操作是昂贵的。**

## `TransportClientFactory`

**`TransportClientFactory` 的 `createClient()` 方法创建` TransportClient`实例**。包含以下重要的成员变量

- `context`: `TransportContext`实例

- `conf`: `TransportConf`实例

- `clientBootstraps`: `List<TransportClientBootstrap>`，在`TransportClient`上执行的客户端引导程序，主要对连接建立时进行一些初始化的准备(例如验证、加密)

- `connectionPool`: `ConcurrentHashMap<SocketAddress, ClientPool>`。**维护到其他远程主机的`TransportClient`连接池映射表**，线程安全

  <img src="{{ site.url }}/assets/img/2020-9-2-3.png" style="zoom:50%;" />

  - 如下所示，`ClientPool`为相同远程主机的`TransportClient`连接池，使其尽量复用。由于线程不安全，所以在放入连接池前，要获取到对应的`lock`

  ```scala
  private static class ClientPool {
    TransportClient[] clients;
  	Object[] locks;

    ClientPool(int size) {
      clients = new TransportClient[size];
      locks = new Object[size];
      for (int i = 0; i < size; i++) {
        locks[i] = new Object();
      }
    }
  }
  ```

- `rand`: `Random`

- `numConnectionsPerPeer`: 一个rpcAddress的连接数

如下所示，`TransportClientFactory` 的 `createClient()` 方法。

- 首先根据远程主机的域名和端口在`connectionPool`取出对应的连接池，接着在连接池中随机取出一个client
- 如果连接池中没有存活的client，则创建一个新的client放入连接池中。**在这一步骤多个线程可能会产生竞态条件，所以要先获得这个连接池的锁**

```java
public TransportClient createClient(String remoteHost, int remotePort)
throws IOException, InterruptedException {
  // Get connection from the connection pool first.
  // If it is not found or not active, create a new one.
  // Use unresolved address here to avoid DNS resolution each time we creates a client.
  final InetSocketAddress unresolvedAddress =
  InetSocketAddress.createUnresolved(remoteHost, remotePort);

  // Create the ClientPool if we don't have it yet.
  ClientPool clientPool = connectionPool.get(unresolvedAddress);
  if (clientPool == null) {
    connectionPool.putIfAbsent(unresolvedAddress, new ClientPool(numConnectionsPerPeer));
    clientPool = connectionPool.get(unresolvedAddress);
  }

  int clientIndex = rand.nextInt(numConnectionsPerPeer);
  TransportClient cachedClient = clientPool.clients[clientIndex];

  if (cachedClient != null && cachedClient.isActive()) {
    // Make sure that the channel will not timeout by updating the last use time of the
    // handler. Then check that the client is still alive, in case it timed out before
    // this code was able to update things.
    TransportChannelHandler handler = cachedClient.getChannel().pipeline()
    .get(TransportChannelHandler.class);
    synchronized (handler) {
      handler.getResponseHandler().updateTimeOfLastRequest();
    }

    if (cachedClient.isActive()) {
      logger.trace("Returning cached connection to {}: {}",
                   cachedClient.getSocketAddress(), cachedClient);
      return cachedClient;
    }
  }

  // If we reach here, we don't have an existing connection open. Let's create a new one.
  // Multiple threads might race here to create new connections. Keep only one of them active.
  final long preResolveHost = System.nanoTime();
  final InetSocketAddress resolvedAddress = new InetSocketAddress(remoteHost, remotePort);
  final long hostResolveTimeMs = (System.nanoTime() - preResolveHost) / 1000000;
  if (hostResolveTimeMs > 2000) {
    logger.warn("DNS resolution for {} took {} ms", resolvedAddress, hostResolveTimeMs);
  } else {
    logger.trace("DNS resolution for {} took {} ms", resolvedAddress, hostResolveTimeMs);
  }

  synchronized (clientPool.locks[clientIndex]) {
    cachedClient = clientPool.clients[clientIndex];

    if (cachedClient != null) {
      if (cachedClient.isActive()) {
        logger.trace("Returning cached connection to {}: {}", resolvedAddress, cachedClient);
        return cachedClient;
      } else {
        logger.info("Found inactive connection to {}, creating a new one.", resolvedAddress);
      }
    }
    clientPool.clients[clientIndex] = createClient(resolvedAddress);
    return clientPool.clients[clientIndex];
  }
}
```

具体创建`TransportClient`是在私有的`createClient()`方法中

- 先构建根引导程序Bootstrap并对其进行配置
- 为根引导程序设置管道初始化回调函数，用于初始化`Channel`的pipeline
- 用客户端引导程序`TransportClientBootstrap`进行初始化

```java
private TransportClient createClient(InetSocketAddress address)
  throws IOException, InterruptedException {
  logger.debug("Creating new connection to {}", address);

  Bootstrap bootstrap = new Bootstrap();
  bootstrap.group(workerGroup)
    .channel(socketChannelClass)
    // Disable Nagle's Algorithm since we don't want packets to wait
    .option(ChannelOption.TCP_NODELAY, true)
    .option(ChannelOption.SO_KEEPALIVE, true)
    .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, conf.connectionTimeoutMs())
    .option(ChannelOption.ALLOCATOR, pooledAllocator);

  if (conf.receiveBuf() > 0) {
    bootstrap.option(ChannelOption.SO_RCVBUF, conf.receiveBuf());
  }

  if (conf.sendBuf() > 0) {
    bootstrap.option(ChannelOption.SO_SNDBUF, conf.sendBuf());
  }

  final AtomicReference<TransportClient> clientRef = new AtomicReference<>();
  final AtomicReference<Channel> channelRef = new AtomicReference<>();

  bootstrap.handler(new ChannelInitializer<SocketChannel>() {
    @Override
    public void initChannel(SocketChannel ch) {
      TransportChannelHandler clientHandler = context.initializePipeline(ch);
      clientRef.set(clientHandler.getClient());
      channelRef.set(ch);
    }
  });

  // Connect to the remote server
  long preConnect = System.nanoTime();
  ChannelFuture cf = bootstrap.connect(address);
  if (!cf.await(conf.connectionTimeoutMs())) {
    throw new IOException(
      String.format("Connecting to %s timed out (%s ms)", address, conf.connectionTimeoutMs()));
  } else if (cf.cause() != null) {
    throw new IOException(String.format("Failed to connect to %s", address), cf.cause());
  }

  TransportClient client = clientRef.get();
  Channel channel = channelRef.get();
  assert client != null : "Channel future completed successfully with null client";

  // Execute any client bootstraps synchronously before marking the Client as successful.
  long preBootstrap = System.nanoTime();
  logger.debug("Connection to {} successful, running bootstraps...", address);
  try {
    for (TransportClientBootstrap clientBootstrap : clientBootstraps) {
      clientBootstrap.doBootstrap(client, channel);
    }
  } catch (Exception e) { // catch non-RuntimeExceptions too as bootstrap may be written in Scala
    long bootstrapTimeMs = (System.nanoTime() - preBootstrap) / 1000000;
    logger.error("Exception while bootstrapping client after " + bootstrapTimeMs + " ms", e);
    client.close();
    throw Throwables.propagate(e);
  }
  long postBootstrap = System.nanoTime();

  logger.info("Successfully created connection to {} after {} ms ({} ms spent in bootstraps)",
              address, (postBootstrap - preConnect) / 1000000, (postBootstrap - preBootstrap) / 1000000);

  return client;
}
```

## `MessageHandler`

<img src="{{ site.url }}/assets/img/2020-9-2-4.png" style="zoom:50%;" />

如上图所示，`MessageHandler`的继承关系

- `handle()`：用于对接收到的单个消息进行处理。
- `channelActive`：当channel激活时调用。
- `exceptionCaught`：当捕获到channel发生异常时调用。
- `channelInactive`：当channel非激活时调用。

`MessageHandler` 用于处理派生自`Message`接口的消息

<img src="{{ site.url }}/assets/img/2020-9-2-5.png"  style="zoom: 50%;" />

- RequestMessage
  - ChunkFetchRequest：请求获取流的单个块的序列。
  - RpcRequest：此消息类型由远程的RPC服务端进行处理，是一种需要服务端向客户端回复的RPC请求信息类型。
  - OneWayMessage：此消息也需要由远程的RPC服务端进行处理，与RpcRequest不同的是，不需要服务端向客户端回复。
  - StreamRequest：此消息表示向远程的服务发起请求，以获取流式数据。由
- ResponseMessage
  - ChunkFetchSuccess：处理ChunkFetchRequest成功后返回的消息。
  - ChunkFetchFailure：处理ChunkFetchRequest失败后返回的消息。
  - RpcResponse：处理RpcRequest成功后返回的消息。
  - RpcFailure：处理RpcRequest失败后返回的消息。
  - StreamResponse：处理StreamRequest成功后返回的消息。
  - StreamFailure：处理StreamRequest失败后返回的消息。

### `TransportRequestHandler`

**server端处理client请求的处理程序**。主要工作方法为`handle()`

```scala
public void handle(RequestMessage request) {
  if (request instanceof ChunkFetchRequest) {
    processFetchRequest((ChunkFetchRequest) request);
  } else if (request instanceof RpcRequest) {
    processRpcRequest((RpcRequest) request);
  } else if (request instanceof OneWayMessage) {
    processOneWayMessage((OneWayMessage) request);
  } else if (request instanceof StreamRequest) {
    processStreamRequest((StreamRequest) request);
  } else if (request instanceof UploadStream) {
    processStreamUpload((UploadStream) request);
  } else {
    throw new IllegalArgumentException("Unknown request type: " + request);
  }
}
```

- `processFetchRequest()`: 处理块获取请求，依赖`StreamManager`获取块
- `processRpcRequest()`: 处理RPC请求，依赖`RpcHandler`的`receive()`方法
- `processOneWayMessage()`: 处理无需回复的RPC请求，依赖`RpcHandler`的`receive()`方法
- `processStreamRequest()`: 处理流请求，依赖`StreamManager`的`openStream()`方法获取流数据并封装成`ManagedBuffer`
- `processStreamUpload()`: 处理流上传请求，依赖`RpcHandler`的`receiveStream()`方法

### `TransportResponseHandler`

**client端处理server响应的处理程序**。

**在client端发送消息时，根据发送消息的类型调用`TransportResponseHandler`中的方法注册回调函数**，回调函数和请求信息放入相应的缓存中。

待`TransportResponseHandler`收到server端的响应消息时，再**调用主要的工作方法`handle()`，根据响应消息类型从对应缓存中取出回调函数并调用**。

## `TransportChannelHandler`

**传输层的handler，负责委托请求给`TransportRequestHandler`，委托响应给`TransportResponseHandler`。**

关键方法`channelRead`负责将请求委托给`TransportRequestHandler`，将响应委托给`TransportResponseHandler`。

```scala
@Override
public void channelRead(ChannelHandlerContext ctx, Object request) throws Exception {
  if (request instanceof RequestMessage) {
    requestHandler.handle((RequestMessage) request);
  } else if (request instanceof ResponseMessage) {
    responseHandler.handle((ResponseMessage) request);
  } else {
    ctx.fireChannelRead(request);
  }
}
```

## `TransportClient`

**用于向server端发送rpc请求和从server 端获取流的chunk块**，一般使用方式如下

```scala
// 打开远程文件
client.sendRPC(new OpenFile("/foo"))
// 获取远程文件的chunk
client.fetchChunk(streamId = 100, chunkIndex = 0, callback)
client.fetchChunk(streamId = 100, chunkIndex = 1, callback)
// 关闭远程文件
client.sendRPC(new CloseStream(100))
```

有两个内部类：`RpcChannelListener`和`StdChannelListener`，继承关系如下

![]({{ site.url }}/assets/img/2020-9-2-1.png)

公共父类`GenericFutureListener `作用是监听一个`Future`对象的执行结果，**通过`Future.addListener(GenericFutureListener)`的方法，这个监听器会在异步任务执行成功之后，调用 `operationComplete` 方法**。`StdChannelListener`实现了`operationComplete()`方法，只是增加了日志。`RpcChannelListener`则在`StdChannelListener`基础上实现了`handleFailure()`方法，失败回调`RpcResponseCallback`的错误处理方法。

下面是`TransportClient`的主要成员变量

- `channel`
- `handler`: `TransportResponseHandler`，server端的请求响应处理器
- `clientId`

下面介绍一下`TransportClient`的两个方法`fetchChunk()`和`sendRpc()`

- `fetchChunk()`: **从远端协商好的流中请求单个块。**调用`TransportResponseHandler`的`addRpcRequest()`方法添加requestId和回调类`RpcResponseCallback`的对应关系，这个回调函数用于处理server端的响应，再通过`Channel.writeAndFlush()`方法将块请求消息发送出去

  ```java
  public void fetchChunk(
    long streamId,
    int chunkIndex,
    ChunkReceivedCallback callback) {
    if (logger.isDebugEnabled()) {
      logger.debug("Sending fetch chunk request {} to {}", chunkIndex, getRemoteAddress(channel));
    }

    StreamChunkId streamChunkId = new StreamChunkId(streamId, chunkIndex);
    StdChannelListener listener = new StdChannelListener(streamChunkId) {
      @Override
      void handleFailure(String errorMsg, Throwable cause) {
        handler.removeFetchRequest(streamChunkId);
        callback.onFailure(chunkIndex, new IOException(errorMsg, cause));
      }
    };
    handler.addFetchRequest(streamChunkId, callback);

    channel.writeAndFlush(new ChunkFetchRequest(streamChunkId)).addListener(listener);
  }
  ```

- `sendRpc()`: **向服务端发送RPC的请求**。首先使用UUID生成唯一的请求id，在`TransportResponseHandler`中注册该请求id的回调函数，再通过`Channel.writeAndFlush()`方法将Rpc消息发送出去

  ```java
  public long sendRpc(ByteBuffer message, RpcResponseCallback callback) {
    if (logger.isTraceEnabled()) {
      logger.trace("Sending RPC to {}", getRemoteAddress(channel));
    }

    long requestId = requestId();
    handler.addRpcRequest(requestId, callback);

    RpcChannelListener listener = new RpcChannelListener(requestId, callback);
    channel.writeAndFlush(new RpcRequest(requestId, new NioManagedBuffer(message)))
      .addListener(listener);

    return requestId;
  }
  ```


## `TransportServer`

提供**高效，低级别流媒体服务的服务器**。

主要方法就是`init()`初始化，在类的构造函数中调用

- 创建初始化Netty服务端时必要的对象，构建服务端根引导程序
- 为根引导程序设置管道初始化回调函数，此回调函数首先设置`TransportServerBootstrap`列表到根引导程序中，然后调用`TransportContext`的`initializePipeline`方法初始化`Channel`的pipeline。
- 绑定监听端口

```java
private void init(String hostToBind, int portToBind) {

  IOMode ioMode = IOMode.valueOf(conf.ioMode());
  EventLoopGroup bossGroup = NettyUtils.createEventLoop(ioMode, 1,
                                                        conf.getModuleName() + "-boss");
  EventLoopGroup workerGroup =  NettyUtils.createEventLoop(ioMode, conf.serverThreads(),
                                                           conf.getModuleName() + "-server");

  PooledByteBufAllocator allocator = NettyUtils.createPooledByteBufAllocator(
    conf.preferDirectBufs(), true /* allowCache */, conf.serverThreads());

  bootstrap = new ServerBootstrap()
    .group(bossGroup, workerGroup)
    .channel(NettyUtils.getServerChannelClass(ioMode))
    .option(ChannelOption.ALLOCATOR, allocator)
    .option(ChannelOption.SO_REUSEADDR, !SystemUtils.IS_OS_WINDOWS)
    .childOption(ChannelOption.ALLOCATOR, allocator);

  this.metrics = new NettyMemoryMetrics(
    allocator, conf.getModuleName() + "-server", conf);

  if (conf.backLog() > 0) {
    bootstrap.option(ChannelOption.SO_BACKLOG, conf.backLog());
  }

  if (conf.receiveBuf() > 0) {
    bootstrap.childOption(ChannelOption.SO_RCVBUF, conf.receiveBuf());
  }

  if (conf.sendBuf() > 0) {
    bootstrap.childOption(ChannelOption.SO_SNDBUF, conf.sendBuf());
  }

  bootstrap.childHandler(new ChannelInitializer<SocketChannel>() {
    @Override
    protected void initChannel(SocketChannel ch) {
      RpcHandler rpcHandler = appRpcHandler;
      for (TransportServerBootstrap bootstrap : bootstraps) {
        rpcHandler = bootstrap.doBootstrap(ch, rpcHandler);
      }
      context.initializePipeline(ch, rpcHandler);
    }
  });

  InetSocketAddress address = hostToBind == null ?
    new InetSocketAddress(portToBind): new InetSocketAddress(hostToBind, portToBind);
  channelFuture = bootstrap.bind(address);
  channelFuture.syncUninterruptibly();

  port = ((InetSocketAddress) channelFuture.channel().localAddress()).getPort();
  logger.debug("Shuffle server started on port: {}", port);
}
```

## 总结

由下图所示，展示了`TransportContext`创建`TransportClientFactory`和`TransportServer`的流程

<img src="{{ site.url }}/assets/img/2020-9-2-2.png" style="zoom:50%;" />

1. `TransportContext`的`createClientFactory`方法创建传输客户端工厂`TransportClientFactory`的实例。在构造`TransportClientFactory`的实例时，还会传递客户端引导程序`TransportClientBootstrap`的列表

   。`TransportClientFactory`内部维护每个Socket地址的连接池

2. 调用`TransportContext`的`createServer`方法创建传输服务端`TransportServer`的实例



如下图所示，展示RPC框架server端处理请求和响应的流程

<img src="{{ site.url }}/assets/img/2020-9-2-6.png" style="zoom:50%;" />



如下图所示，展示RPC框架client端请求和响应的流程

<img src="{{ site.url }}/assets/img/2020-9-2-7.png"  style="zoom:50%;" />

## REFERENCE

1. [spark 源码分析](https://www.cnblogs.com/johnny666888/p/11259944.html)
2. Spark内核设计的艺术：架构设计与实现









