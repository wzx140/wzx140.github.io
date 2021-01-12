---
layout: post
title: Executor框架与线程池
categories: Java
date:   2021-1-12
keywords: ThreadPoolExecutor, Future, FutureTask, Executor
mathjax: false
author: wzx
---


介绍Executor框架的整体结构，主要分析了线程池的工作原理




## Executor框架

在上层，Java多线程程序通常把应用分解为若干个任务，然后使用Executor框架将这些任务映射为固定数量的线程。在底层，操作系统内核将这些线程映射到硬件处理器上。

如果所示，Executor框架的架构

![]({{ site.url }}/assets/img/2021-1-12-1.png)

### 任务

**被执行任务需要实现`Runnale`或者`Callable`接口，才可以提交给`ExecutorService`子类执行**，`Runable`与`Callable`的区别在于是否有返回值

### ExecutorService

#### ThreadPoolExecutor

具体原理下一节介绍，这里介绍几种常见的线程池

- `ThreadPoolExecutor`: **基础线程池**
  - `corePoolSize`: **线程池中的线程数量**，如果运行的线程少于 `corePoolSize`，则创建新线程来处理任务，即使线程池中的其他线程是空闲的
  - `maximumPoolSize`: **线程池中的最大线程数量**
  - `keepAliveTime` : 当线程池线程数量超过`corePoolSize`时，**多余的空闲线程的存活时间**
  - `unit`: `keepAliveTime`的单位
  - `workQueue` : **阻塞队列**，被提交但尚未被执行的任务
  - `threadfactory`: **线程工厂**
  - `handler`: **拒绝策略**


- `SingleThreadExecutor`: **单线程池**，适用于顺序执行各个任务
  - `Executors.newSingleThreadExecutor()`
  - `new ThreadPoolExecutor(1, 1, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>())`
- `FixedThreadPool`: **固定线程池**，适用于负载较重的服务器，限制线程数量满足资源需求
  - `Executors.newFixedThreadPool(int nThreads)`
  - `new ThreadPoolExecutor(nThreads, nThreads, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>())`
- `CachedThreadPool`: **无穷大的线程池**，适用于较多的短期异步任务
  - `Executors.newCachedThreadPool()`
  - `new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60L, TimeUnit.SECONDS, new SynchronousQueue<Runnable>())`

#### ScheduledThreadPoolExecutor

通过`scheduleAtFixedRate()`设置初始执行延迟和`	scheduleWithFixedDelay()`设置任务间延迟的线程池

![]({{ site.url }}/assets/img/2021-1-12-2.png)

### Future

- `Future`：保存任务执行的结果，对任务进行**取消、查询状态、获取结果**
- `FutureTask`：`Future`接口与`Runable`接口的实现类，两者功能的结合

`FutureTask`有以下7种状态

```java
/**
 * 在构建FutureTask时设置，同时也表示内部成员callable已成功赋值
 * 一直到worker thread完成FutureTask中的run()
 */
private static final int NEW          = 0;
/**
 * worker thread在处理task时设定的中间状态，处于该状态时，说明worker thread正准备设置result
 */
private static final int COMPLETING   = 1;
/**
 * 当设置result结果完成后，FutureTask处于该状态，代表过程结果。
 * 该状态为最终状态final state。（正确完成的最终状态）
 */
private static final int NORMAL       = 2;
/**
 * 同上，只不过task执行过程出现异常，此时结果设置为exception，也是final state
 */
private static final int EXCEPTIONAL  = 3;
/**
 * final state，表明task被cancel（task还没有执行就被cancel的状态）
 */
private static final int CANCELLED    = 4;
/**
 * 中间状态，task运行过程中被interrupt时，设置的中间状态
 */
private static final int INTERRUPTING = 5;
/**
 * final state，中断完毕的最终状态
 */
private static final int INTERRUPTED  = 6;
```

`FutureTask`拥有以下属性

```java
// 用来执行的任务
private Callable<V> callable;
// 用来保存任务的成功执行的结果后者执行任务时发生异常的Exception对象
private Object outcome; 
// 执行Callable任务的线程
private volatile Thread runner;
// 调用get方法获取任务执行结果时被阻塞的线程栈
private volatile WaitNode waiters;

// 维护等待在此FutureTask上的线程, volatile保证了线程间的可见性, 因为需要CAS操作
static final class WaitNode {
    volatile Thread thread;
    volatile WaitNode next;
    WaitNode() { thread = Thread.currentThread(); }
}
```

`FutureTask`实例提交给`ExecutorService`后，在`Worker`中调用`FutureTask.run()`来执行任务

```java
public void run() {
  if (state != NEW ||
      !UNSAFE.compareAndSwapObject(this, runnerOffset, null, Thread.currentThread()))
    return;
  try {
    Callable<V> c = callable;
    if (c != null && state == NEW) {
      V result;
      boolean ran;
      try {
        result = c.call();
        ran = true;
      } catch (Throwable ex) {
        result = null;
        ran = false;
        setException(ex);
      }
      if (ran)
        set(result);
    }
  } finally {
    runner = null;
    int s = state;
    if (s >= INTERRUPTING)
      handlePossibleCancellationInterrupt(s);
  }
}
```

- 如果任务状态不是`NEW`，表示任务已经执行过，如果CAS设置`Runner`失败，表示有其他线程正在执行此任务，这两种情况可以直接返回
- 调用`Callable.call()`执行任务
- 调用`set()`方法设置结果，并调用了`finishCompletion()`方法，唤醒所有`WaitNode`

`FutureTask`的`get()`方法用于阻塞直到获取任务执行结果

```java
public V get() throws InterruptedException, ExecutionException {
  int s = state;
  if (s <= COMPLETING)
    // 阻塞直到COMPLETING或中断
    s = awaitDone(false, 0L);
  return report(s);
}

private int awaitDone(boolean timed, long nanos)
  throws InterruptedException {
  final long deadline = timed ? System.nanoTime() + nanos : 0L;
  WaitNode q = null;
  boolean queued = false;
  // 线程自旋
  for (;;) {
    // 响应中断
    if (Thread.interrupted()) {
      removeWaiter(q);
      throw new InterruptedException();
    }

    int s = state;
    // 返回
    if (s > COMPLETING) {
      if (q != null)
        q.thread = null;
      return s;
    }
    // 让出时间片给工作线程
    else if (s == COMPLETING)
      Thread.yield();
    else if (q == null)
      q = new WaitNode();
    // 添加线程到waiters等待队列, 完成后唤醒
    else if (!queued)
      queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                           q.next = waiters, q);
    else if (timed) {
      nanos = deadline - System.nanoTime();
      if (nanos <= 0L) {
        removeWaiter(q);
        return state;
      }
      LockSupport.parkNanos(this, nanos);
    }
    else
      LockSupport.park(this);
  }
}
```

## ThreadPoolExecutor

线程池由**`ctl`变量维护了运行状态和运行线程数量**这两个状态，有以下五种运行状态

- `RUNNING`: 能接受新提交的任务，并且也能处理阻塞队列中的任务。
- `SHUTDOWN`: 不再接受新提交的任务，但却可以继续处理阻塞队列中已保存的任务
- `STOP`: 不能接受新任务，也不处理队列中的任务，会中断正在处理任务的线程
- `TIDYING`: 运行线程数量为0
- `TERMINATED`: 在`terminated()`方法执行完后进入该状态

![]({{ site.url }}/assets/img/2021-1-12-3.png)

线程池应用了**生产者消费者模式**，通过阻塞队列来缓存暂时不能执行的任务

![]({{ site.url }}/assets/img/2021-1-12-4.png)

### Worker

线程池通过内部类`Worker`掌握线程的状态并维护线程的生命周期。如下所示，`Worker`维护了一个**`thread`用来执行任务，和`firstTask`保存传入的第一个任务**。

```java
private final class Worker extends AbstractQueuedSynchronizer implements Runnable{
  final Thread thread;
  Runnable firstTask;
  ...
}
```

通过调用`Worker.thread.start()`启动工作线程，调用栈为`Worker.thread.start()`->`Worker.run()`->`Worker.runWorker(this)`，源码如下

```java
final void runWorker(Worker w) {
  Thread wt = Thread.currentThread();
  Runnable task = w.firstTask;
  w.firstTask = null;
  w.unlock(); // allow interrupts
  boolean completedAbruptly = true;
  try {
    while (task != null || (task = getTask()) != null) {
      w.lock();
      if ((runStateAtLeast(ctl.get(), STOP) ||
           (Thread.interrupted() &&
            runStateAtLeast(ctl.get(), STOP))) &&
          !wt.isInterrupted())
        wt.interrupt();
      try {
        beforeExecute(wt, task);
        Throwable thrown = null;
        try {
          task.run();
        } catch (RuntimeException x) {
          thrown = x; throw x;
        } catch (Error x) {
          thrown = x; throw x;
        } catch (Throwable x) {
          thrown = x; throw new Error(x);
        } finally {
          afterExecute(task, thrown);
        }
      } finally {
        task = null;
        w.completedTasks++;
        w.unlock();
      }
    }
    completedAbruptly = false;
  } finally {
    processWorkerExit(w, completedAbruptly);
  }
}
```

1. While循环不断地通过`getTask()`方法**从阻塞队列中获取任务**
2. 加锁，`Worker`是通过继承AQS实现了锁的功能。**通过当前`Worker`是否持有锁从而判断Worker是否在执行任务还是阻塞在`getTask()`**
3. 如果线程池处于`STOP`状态，保证当前线程是中断状态，否则要保证当前线程不是中断状态
4. 执行任务
5. `getTask()`返回null时跳出循环，执行`processWorkerExit()`方法销毁线程
   - `Worker`数量超过`maximumPoolSize`
   - 线程池处于`STOP`状态
   - 线程池处于`SHUTDOWN`并且阻塞队列为空
   - 阻塞队列的`poll()`方法超时

### execute()

`execute()`方法用于向线程池中提交任务，源码如下所示

```scala
public void execute(Runnable command) {
  if (command == null)
  	throw new NullPointerException();
  int c = ctl.get();
  if (workerCountOf(c) < corePoolSize) {
    // true: 使用corePoolSize作为边界
    if (addWorker(command, true))
    return;
    c = ctl.get();
  }
  if (isRunning(c) && workQueue.offer(command)) {
    int recheck = ctl.get();
    if (! isRunning(recheck) && remove(command))
    reject(command);
    else if (workerCountOf(recheck) == 0)
    // false: 使用maximumPoolSize作为边界
    addWorker(null, false);
  }
  else if (!addWorker(command, false))
  reject(command);
}
```

1. 如果**当前运行的线程少于`corePoolSize`**，则获得全局锁并创建新线程来执行任务
2. 如果**运行的线程等于或多于`corePoolSize`**，则将任务加入`Blockingqueue`
3. **`Blockingqueue`已满**，则创建新的线程来处理任务
4. 如果**当前运行的线程超出`maximumPoolSize`**，则触发拒绝策略

![]({{ site.url }}/assets/img/2021-1-12-5.png)

### shutDown()

- `shutdown()`: 只会告诉执行者服务它**不能接受新任务**，但是已经提交的任务将继续运行
- `shutdownNow()`: 将执行相同的操作，并尝试**通过中断相关线程来取消已提交的任务**

### 拒绝策略

#### CallerRunsPolicy

**调用者运行策略**。当触发拒绝策略时，只要线程池没有关闭，就**由提交任务的当前线程执行任务**。

```java
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
  if (!e.isShutdown()) {
    r.run();
  }
}
```

一般在**不允许失败的、对性能要求不高、并发量较小的场景**下使用，因为线程池一般情况下不会关闭，也就是提交的任务一定会被运行，但是由于是调用者线程自己执行的，当多次提交任务时，就会**阻塞后续任务执行，性能和效率自然就慢了**。

#### AbortPolicy

**中止策略，默认拒绝策略**。当触发拒绝策略时，**直接抛出拒绝执行的异常**。

```java
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
  throw new RejectedExecutionException("Task " + r.toString() +
                                       " rejected from " +
                                       e.toString());
}
```

#### DiscardPolicy

**丢弃策略**。**直接丢弃这个任务**，不触发任何动作。如果你提交的任务无关紧要，你就可以使用它 。因为它就是个空实现，会悄无声息的吞噬你的的任务。

```java
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
}
```

#### DiscardOldestPolicy

**弃老策略**。如果线程池未关闭，就**弹出阻塞队列头部的元素(丢弃老任务)，然后尝试重新提交**。

```java
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
  if (!e.isShutdown()) {
    e.getQueue().poll();
    e.execute(r);
  }
}
```

#### NewThreadRunsPolicy

**新线程运行策略**。执行实例化一个新线程去执行任务

```java
public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
  try {
    Thread t = new Thread(r, "Temporary task executor");
    t.start();
  } catch (Throwable var4) {
    throw new RejectedExecutionException("Failed to start a new thread", var4);
  }
}
```

### 优势

- **降低资源消耗**。通过重复利用已创建的线程降低线程创建和销毁造成的消耗
- **提高响应速度**。当任务到达时，任务可以不需要等到线程创建就能立即执行
- **提高线程的可管理性**。线程是稀缺资源，使用线程池可以进行统一分配、调优和监控

### 合理配置

- **CPU密集型任务应配置尽可能小的线程**，任务数尽可能接近核心数
- **IO密集型任务线程应配置尽可能多的线程**，执行任务不是一直使用核心
- **给任务规定优先级**，使用`PriorityBlockingQueue`作为阻塞队列
- **建议使用有界队列**，大量任务的挤压会导致OOM

## REFERENCE

1. Java并发编程的艺术
2. [Java线程池实现原理及其在美团业务中的实践](https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html)
3. [FutureTask原理分析](https://www.youendless.com/post/futuretask/)