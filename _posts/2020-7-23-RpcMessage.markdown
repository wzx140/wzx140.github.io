---
layout: post
title:  "Spark源码阅读(二)：RPC之Inbox、Outbox、Dispatcher"
date:   2020-7-23
categories: Spark
keywords: Spark, RPC
mathjax: false
author: wzx
---


Spark RPC中发送消息和接收消息的底层分发处理





## `InboxMessage`&`Inbox`

**`InboxMessage`是一个特质，所有的RPC消息都继承自`InboxMessage`**。下面是继承自`InboxMessage`的子类

- `OneWayMessage`：`RpcEndpoint`处理此类型的消息后不需要向客户端回复信息。

- `RpcMessage`：`RpcEndpoint`处理完此消息后需要向客户端回复信息。

- `OnStart`：`Inbox`实例化后，再通知与此`Inbox`相关联的`RpcEndpoint`启动。

- `OnStop`：`Inbox`停止后，通知与此`Inbox`相关联的`RpcEndpoint`停止。

- `RemoteProcessConnected`：告诉所有的`RpcEndpoint`，有远端的进程已经与当前RPC服务建立了连接。

- `RemoteProcessDisconnected`：告诉所有的`RpcEndpoint`，有远端的进程已经与当前RPC服务断开了连接。

- `RemoteProcessConnectionError`：告诉所有的`RpcEndpoint`，与远端某个地址之间的连接发生了错误。



**`Inbox`为`RpcEndpoint`存储了消息即`InboxMessage`，并线程安全地发送给`RpcEndPoint`**。下面是重要的属性

- `messages`：**所有的消息以链表地形式存储**
- `enableConcurrent`：**是否同时允许多线程**
- `numActiveThreads`：**正在处理这个box的线程数**

下面是`Inbox`的重要方法

- `post()`：**将`InboxMessage`投递到box中**，从下面的代码可以看出使用了`synchronized`保证线程安全，如果该box已经关闭，消息将会丢弃

  ```scala
  def post(message: InboxMessage): Unit = inbox.synchronized {
      if (stopped) {
          // We already put "OnStop" into "messages", so we should drop further messages
          onDrop(message)
      } else {
          messages.add(message)
          false
      }
  }
  ```

- `process()`：**处理存储在`messages`中的消息**。

  - 首先进行并发检查，取出`message`并将`numActiveThreads`自增，因为`LinkedList`线程不安全所以使用了同步
  - 利用模式匹配，对不同的消息类型调用`endpoint`的不同回调函数
  - `safelyCall()`用于统一的异常处理
  - 当存在其他活跃线程或者消息已经处理完则退出当前循环，否则继续处理消息
  
  ```scala
  /**
     * Calls action closure, and calls the endpoint's onError function in the case of exceptions.
     */
  private def safelyCall(endpoint: RpcEndpoint)(action: => Unit): Unit = {
      try action catch {
          case NonFatal(e) =>
          try endpoint.onError(e) catch {
              case NonFatal(ee) =>
              if (stopped) {
                  logDebug("Ignoring error", ee)
              } else {
                  logError("Ignoring error", ee)
              }
          }
      }
  }
  
  def process(dispatcher: Dispatcher): Unit = {
      var message: InboxMessage = null
      inbox.synchronized {
          if (!enableConcurrent && numActiveThreads != 0) {
              return
          }
          message = messages.poll()
          if (message != null) {
              numActiveThreads += 1
          } else {
              return
          }
      }
      while (true) {
          safelyCall(endpoint) {
              message match {
                  case RpcMessage(_sender, content, context) =>
                  try {
                      endpoint.receiveAndReply(context).applyOrElse[Any, Unit](content, { msg =>
                          throw new SparkException(s"Unsupported message $message from ${_sender}")
                      })
                  } catch {
                      case e: Throwable =>
                      context.sendFailure(e)
                      // Throw the exception -- this exception will be caught by the safelyCall function.
                      // The endpoint's onError function will be called.
                      throw e
                  }
  
                  case OneWayMessage(_sender, content) =>
                  endpoint.receive.applyOrElse[Any, Unit](content, { msg =>
                      throw new SparkException(s"Unsupported message $message from ${_sender}")
                  })
  
                  case OnStart =>
                  endpoint.onStart()
                  if (!endpoint.isInstanceOf[ThreadSafeRpcEndpoint]) {
                      inbox.synchronized {
                          if (!stopped) {
                              enableConcurrent = true
                          }
                      }
                  }
  
                  case OnStop =>
                  val activeThreads = inbox.synchronized { inbox.numActiveThreads }
                  assert(activeThreads == 1,
                         s"There should be only a single active thread but found $activeThreads threads.")
                  dispatcher.removeRpcEndpointRef(endpoint)
                  endpoint.onStop()
                  assert(isEmpty, "OnStop should be the last message")
  
                  case RemoteProcessConnected(remoteAddress) =>
                  endpoint.onConnected(remoteAddress)
  
                  case RemoteProcessDisconnected(remoteAddress) =>
                  endpoint.onDisconnected(remoteAddress)
  
                  case RemoteProcessConnectionError(cause, remoteAddress) =>
                  endpoint.onNetworkError(cause, remoteAddress)
              }
          }
  
          inbox.synchronized {
              // "enableConcurrent" will be set to false after `onStop` is called, so we should check it
              // every time.
              if (!enableConcurrent && numActiveThreads != 1) {
                  // If we are not the only one worker, exit
                  numActiveThreads -= 1
                  return
              }
              message = messages.poll()
              if (message == null) {
                  numActiveThreads -= 1
                  return
            }
          }
    }
  }
  ```
  
- `stop()`：`enableConcurrent`赋值为false，保证当前是唯一活跃的线程。并在`messages`中添加`onStop`消息

  ```scala
  def stop(): Unit = inbox.synchronized {
      // The following codes should be in `synchronized` so that we can make sure "OnStop" is the last
      // message
      if (!stopped) {
          // We should disable concurrent here. Then when RpcEndpoint.onStop is called, it's the only
          // thread that is processing messages. So `RpcEndpoint.onStop` can release its resources
          // safely.
          enableConcurrent = false
          stopped = true
          messages.add(OnStop)
          // Note: The concurrent events in messages will be processed one by one.
      }
  }
  ```


## `Outbox`&`OutboxMessage`

**`OutboxMessage`在客户端使用，是对外发送消息的封装。`InboxMessage`在服务端使用，是对所接收消息的封装**。`OutboxMessage`是一个特质，内部只有未实现的`SendWith`方法和`onFailure`方法。`OneWayOutboxMessage`和`RpcOutboxMessage`都继承自`OutboxMessage`特质，**实现的`SendWith`通过调用`TransportClient`的`sendRpc()`方法发送信息**，其中`RpcOutboxMessage`还增加了超时和发送成功的回调方法。



`Outbox`包含以下重要的成员变量
- `messages`: 保存要发送的`OutboxMessage`。`LinkedList`类型，线程不安全
- `client`: `TransportClient`
- `stopped`: 当前`Outbox`是否停止的标识
- `draining`: 表示当前`Outbox`内正有线程在处理`messages`中消息的状态 

如以下代码所示的重要的方法，**之所以要使用以下这种机制来发消息，是保证并发发送消息时，所有消息依次添加到`Outbox`中，并依次传输，同时不会阻塞`send()`方法**
- `send()`：将要发送的`OutboxMessage`首先保存到成员变量链表`messages`中，若`Outbox`未停止则调用`drainOutbox()`方法处理`messages`中的信息。**因为`messages`是`LinkedList`类型，线程不安全，所以在添加和删除时使用了同步机制**。之后调用了私有的`drainOutbox()`方法发送消息

  ```scala
  def send(message: OutboxMessage): Unit = {
    val dropped = synchronized {
      if (stopped) {
        true
      } else {
        messages.add(message)
        false
      }
    }
    if (dropped) {
      message.onFailure(new SparkException("Message is dropped because Outbox is stopped"))
    } else {
      drainOutbox()
    }
  }
  ```

- `drainOutbox()`: 先判断是否已停止，client是否空等前置条件。取出一条消息，并将`draining`置为true，接下来将`messages`中所有消息调用`sendWith()`方法发送。

  ```scala
  private def drainOutbox(): Unit = {
    var message: OutboxMessage = null
    synchronized {
      if (stopped) {
        return
      }
      if (connectFuture != null) {
        // We are connecting to the remote address, so just exit
        return
      }
      if (client == null) {
        // There is no connect task but client is null, so we need to launch the connect task.
        launchConnectTask()
        return
      }
      if (draining) {
        // There is some thread draining, so just exit
        return
      }
      message = messages.poll()
      if (message == null) {
        return
      }
      draining = true
    }
    while (true) {
      try {
        val _client = synchronized { client }
        if (_client != null) {
          message.sendWith(_client)
        } else {
          assert(stopped == true)
        }
      } catch {
        case NonFatal(e) =>
        handleNetworkFailure(e)
        return
      }
      synchronized {
        if (stopped) {
          return
        }
        message = messages.poll()
        if (message == null) {
          draining = false
          return
        }
      }
    }
  }
  ```

- `launchConnectTask()`: 初始化`client`

- `stop()`: 停止`Outbox`
  - 将`Outbox`的停止状态stopped置为true
  - 关闭`TransportClient`
  - 清空`messages`中的消息
  ```scala
  def stop(): Unit = {
    synchronized {
      if (stopped) {
        return
      }
      stopped = true
      if (connectFuture != null) {
        connectFuture.cancel(true)
      }
      closeClient()
    }

    // We always check `stopped` before updating messages, so here we can make sure no thread will
    // update messages and it's safe to just drain the queue.
    var message = messages.poll()
    while (message != null) {
      message.onFailure(new SparkException("Message is dropped because Outbox is stopped"))
      message = messages.poll()
    }
  }
  ```

## `RequestMessage`

**保存从发送方向接收方传递的任意类型的消息**。此外还保存有发送方地址`senderAddress: RpcAddress`，接收方的远程`RpcEndpoint`引用`receiver: NettyRpcEndpointRef`，实现了上述参数的序列化。

## `Dispatcher`

**`Dispatcher`负责将RPC消息路由到要该对此消息处理的`RpcEndpoint`**。

下面是一些重要的属性。

- `endpoints`：**储存`name`和`EndpointData`的映射关系**。`EndpointData`包含了`name`，`RpcEndpoint`, `NettyRpcEndpointRef`和`Inbox`，采用`ConcureentHashMap`保证线程安全

- `endpointRefs`：**储存`RpcEndpoint`和`RpcEndpointRef`的映射关系**。采用`ConcureentHashMap`保证线程安全

- `receivers`：**存储`inbox`中可能包含message的`EndpointData`**。在`MessageLoop`中取出并处理消息。使用阻塞队列`LinkedBlockingQueue`存储。

- `threadpool`：**用于调度消息的线程池**。根据`spark.rpc.netty.dispatcher.numThreads`创建固定大小的线程池，启动与线程池大小相同的`MessageLoop`任务。

  ```scala
  private val threadpool: ThreadPoolExecutor = {
      val availableCores =
      if (numUsableCores > 0) numUsableCores else Runtime.getRuntime.availableProcessors()
      val numThreads = nettyEnv.conf.getInt("spark.rpc.netty.dispatcher.numThreads",
                                            math.max(2, availableCores))
      val pool = ThreadUtils.newDaemonFixedThreadPool(numThreads, "dispatcher-event-loop")
      for (i <- 0 until numThreads) {
          pool.execute(new MessageLoop)
      }
      pool
  }
  
  // ThreadUtils
  def newDaemonFixedThreadPool(nThreads: Int, prefix: String): ThreadPoolExecutor = {
      val threadFactory = namedThreadFactory(prefix)
      Executors.newFixedThreadPool(nThreads, threadFactory).asInstanceOf[ThreadPoolExecutor]
  }
  ```
  - `MessageLoop`：**将消息交给`EndpointData`的`Inbox`处理**。实现了`Runnable`接口，从`receivers`中取出`EndpointData`并调用`EndpointData`的`Inbox`的`process()`方法，直到遇到`PoisonPill`哨兵。
    
    ```scala
    // MessageLoop
    private class MessageLoop extends Runnable {
      override def run(): Unit = {
        try {
          while (true) {
            try {
              val data = receivers.take()
              if (data == PoisonPill) {
                // Put PoisonPill back so that other MessageLoops can see it.
                receivers.offer(PoisonPill)
                return
              }
              data.inbox.process(Dispatcher.this)
            } catch {
              case NonFatal(e) => logError(e.getMessage, e)
            }
          }
        } catch {
          case _: InterruptedException => // exit
          case t: Throwable =>
          try {
            // Re-submit a MessageLoop so that Dispatcher will still work if
            // UncaughtExceptionHandler decides to not kill JVM.
            threadpool.execute(new MessageLoop)
          } finally {
            throw t
          }
        }
      }
    }
    ```

下面是一些重要的方法。

- `registerRpcEndpoint()`：**在调度器中注册endpoint**。由`name`和`RpcEndpoint`构建`NettyRpcEndpointRef`，并加入到`endpoints`, `endpointRefs`, `receivers`中

- `postToAll()`：**将message投递到在注册到该`Dispatcher`的所有`RpcEndpoint`**。`postMessage()`将message投递到注册到该`Dispatcher`指定name的`RpcEndpoint`中，并将`EndpointData`放入`receivers`中，该方法中还传入了失败回调函数

  ```scala
  def postToAll(message: InboxMessage): Unit = {
      val iter = endpoints.keySet().iterator()
      while (iter.hasNext) {
          val name = iter.next
          postMessage(name, message, (e) => { e match {
              case e: RpcEnvStoppedException => logDebug (s"Message $message dropped. ${e.getMessage}")
              case e: Throwable => logWarning(s"Message $message dropped. ${e.getMessage}")
          }}
                     )}
  }
  
  private def postMessage(
      endpointName: String,
      message: InboxMessage,
      callbackIfStopped: (Exception) => Unit): Unit = {
      val error = synchronized {
          val data = endpoints.get(endpointName)
          if (stopped) {
              Some(new RpcEnvStoppedException())
          } else if (data == null) {
              Some(new SparkException(s"Could not find $endpointName."))
          } else {
              data.inbox.post(message)
              receivers.offer(data)
              None
          }
      }
      // We don't need to call `onStop` in the `synchronized` block
      error.foreach(callbackIfStopped)
  }
  ```
  - `unregisterRpcEndpoint()`, `stop()`：注销所有已注册的`RpcEndpoint`，从`endpoints`中移除并在`inbox`中增加了`onstop`消息。在`receivers`中插入哨兵，等待`receivers`中的所有消息都处理完毕后，关闭线程池。

  ```scala
  private def unregisterRpcEndpoint(name: String): Unit = {
      val data = endpoints.remove(name)
      if (data != null) {
          data.inbox.stop()
          receivers.offer(data)  // for the OnStop message
      }
      // Don't clean `endpointRefs` here because it's possible that some messages are being processed
      // now and they can use `getRpcEndpointRef`. So `endpointRefs` will be cleaned in Inbox via
      // `removeRpcEndpointRef`.
  }
  
  def stop(): Unit = {
      synchronized {
          if (stopped) {
              return
          }
          stopped = true
      }
      // Stop all endpoints. This will queue all endpoints for processing by the message loops.
      endpoints.keySet().asScala.foreach(unregisterRpcEndpoint)
      // Enqueue a message that tells the message loops to stop.
      receivers.offer(PoisonPill)
      threadpool.shutdown()
  }
  ```


## 总结

![]({{ site.url }}/assets/img/2020-7-23-4.png)

如上图所示，`Dispatcher`中的消息处理流程。

1. `postToAll()`或者`postxx()`方法会调用`postMessage()`方法将`InboxMessage`放到对应`endPointData`里`inbox`的`messages`列表(调用`inbox.post()`)
2. `InboxMessage`放入后`inbox`后，`inbox`所属的`endPointData`就会放入`receivers`
3. 一旦`receivers`中有数据，原本阻塞的`MessageLoop`就可以取到数据，因为`receivers`是一个阻塞队列
4. `MessageLoop`将调用`inbox.process()`方法消息的处理。利用模式匹配，对不同的消息类型调用`endpoint`的不同回调函数，即完成了消息的处理。

## REFERENCE

1. [spark 源码分析](https://www.cnblogs.com/johnny666888/p/11259944.html)
2. Spark内核设计的艺术：架构设计与实现
