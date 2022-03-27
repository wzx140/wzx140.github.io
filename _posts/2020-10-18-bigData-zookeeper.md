---
layout: post
title:  "大数据日知录学习(六)：Zookeeper"
date:   2020-10-18
categories: 大数据理论与架构
keywords: Zookeeper
mathjax: true
author: wzx
---

Yahoo 开源的可拓展高吞吐分布式协调系统。ZK本质是特殊的FS，但用于存储元数据，应用数据需要单独存储





## 体系结构

ZooKeeper服务由若干台结点组成，**每个结点内存中维护相同的类似于文件系统的属性数据结构**，**其中一个结点通过ZAB原子广播协议被选举Leader，其他作为Learner(Follower和Observer)**。client通过TCP协议连接任意一个结点进行读写操作，但如果是Learner最终会由Leader进行协调更新。**ZooKeeper 的设计目标是将那些复杂且容易出错的分布式一致性服务封装起来，构成一个高效可靠的原语集，并以一系列简单易用的接口提供给用户使用。**

<img src="{{ site.url }}/assets/img/2020-10-18-1.png" style="zoom:67%;" />

具有以下特点

- **顺序一致性:** 从同一客户端发起的事务请求，最终将会严格地按照顺序被应用到ZooKeeper集群中去。
- **原子性:** 所有事务请求的处理结果在整个集群中所有机器上的应用情况是一致的，也就是说，要么整个集群中所有的机器都成功应用了某一个事务，要么都没有应用。
- **单一系统映像:** 无论客户端连到哪一个ZooKeeper服务器上，其看到的服务端数据模型都是一致的。
- **可靠性:**  一旦一次更改请求被应用，更改的结果就会被持久化，直到被下一次更改覆盖
- **高性能:** **ZooKeeper将数据保存在内存中，这也就保证了高吞吐量和低延迟**。由于写操作会导致集群机器同步状态，而读操作可以通过增加Observer机器提升读性能，**适用于读多于写的应用程序**
- **高可用:** 半数以上的机器存活服务就能正常运行

## ZAB协议

**集群间通过ZAB协议(*Zookeeper Atomic Broadcast*)来保持数据的一致性及顺序性，是一种支持崩溃恢复的原子广播协议**。Leader将所有更新操作序列化利用 ZAB 协议将数据更新请求通知所有从属服务器，保证Learner的数据更新顺序和Leader的更新顺序是一样的。

如下表所示，ZooKeeper 集群中的**所有机器通过一个 Leader 选举过程来选定一台机器作为Leader**，Leader 既可以为client提供写服务又能提供读服务。除了Leader外，Follower和Observer都只能提供读服务。**Follower和Observer唯一的区别在于 Observer 机器不参与 Leader 的选举过程，也不参与写操作的“过半写成功”策略**，因此Observer机器可以在不影响写性能的情况下提升集群的读性能。

<img src="{{ site.url }}/assets/img/2020-10-18-2.png" style="zoom:67%;" />

ZAB一致性协议采用简单的多数投票仲裁方式，这意味着半数以上投票服务器存活，Zookeeper 才能正常运行，即如果有 2f+1 台服务器，则最多可以容忍f台服务器产生故障。

### Serverid

服务器编号，编号越大在选择算法中的权重越大。

### Zxid

**对于来自client的每个更新请求，ZooKeeper 都会分配一个全局唯一的递增编号Zxid(Zookeeper Transaction Id)，这个编号反应了所有事务操作的先后顺序**，表示服务器中存放的最大数据ID。实际上是一个64位的数字，高32位是Epoch用来标识 leader是否发生改变，低32位用来递增计数。值越大在选择算法中的权重越大。

如果client从某个服务器切换连接到了另外一个服务器，新服务器会保证给这个client看到的数据版本不会比之前的服务器数据版本更低、这是通过比较client发送请求时传来的Zxid和服务器本身的最高编号Zxid来实现的。如果client请求Zxid编号高于服务器本身最高Zxid编号，说明服务器数据过时，则其从Leader同步内存数据到最新状态，然后再响应读操作。

### Epoch

逻辑时钟，表示投票次数。同一轮投票过程中的Epoch是相同的，**每投完一次票Epoch就会增加**。

### 服务器状态

- LOOKING: 竟选状态
- FOLLOWING: 随从状态，同步Leader状态，参与投票 
- OBSERVING: 观察状态，同步Leader状态，不参与投票
- LEADING: 领导者状态

### 崩溃恢复

**当Leader出现网络中断、崩溃退出与重启等异常情况**时，ZAB协议就会进人恢复模式并选举产生新的Leader。当选举产生了新的 Leader 服务器，同时集群中已经有过半的机器与该Leader服务器完成了状态同步之后，ZAB协议就会退出恢复模式。**恢复模式主要进行Leader选举和数据恢复**。

Leader宕机时，剩下的所有Follower都会变成LOOKING状态，然后进行Leader选举流程。每个Server第一轮都会投票给自己，**投票信息为(Serverid, Zxid, Epoch)**。

- 发送的Epoch大于目前的Epoch
  - 更新本Epoch，同时清空本轮Epoch收集到的来自其他Server的选举数据
  - 根据Zxid->Leader Serverid的顺序，判断是否需要更新自己的投票信息
  - 将自身的投票信息广播给其他Server
- 发送的Epoch小于目前的Epoch
  - 直接发送自身的投票信息
- 发送的Epoch等于目前的Epoch
  - 根据Zxid->Leader Serverid的顺序，判断是否需要更新自己的投票信息
  - 将自身的投票信息广播给其他Server
- 已经收集到了半数以上服务器的选举状态，等待200ms确保没有丢失其他Server的更优选票，之后根据投票结果设置自己的角色(FOLLOWING, LEADER)，退出选举

Zookeeper通过**日志重放和模糊快照来对服务器故障进行容错**。**重放日志在将更新操作体现在内存数据之前先写入外存日志中避免数据丢失，模糊快照指周期性不加锁状态下对内存数据做数据快照**。因为Zookeeper可以保证数据更新操作是**幂等**的，所以在服务器故障恢复时，**加载模糊快照并根据重放日志重新应用更新操作**，系统就会恢复到最新状态

### 消息广播

Leader将Client的事务请求分发给集群中其他所有的Follower，然后Leader等待Follower反馈，**当有过半数的Follower反馈信息后，Leader将再次向集群内Follower广播完成事务提交**。**Follower接收到Client的事务请求，会首先将这个事务请求转发给Leader**。ZAB协议的核心就是只要有一台服务器提交Proposal，就要确保所有的服务器最终都能正确提交Proposal。并且Leader服务器与每一个 Follower服务器之间都维护了一个单独的FIFO消息队列进行收发消息，使用队列消息可以做到**异步解耦**。

如下流程所示，相当于简化版的2PC，只要半数以上的Follower成功反馈即可提交

1. Client发起一个写操作请求
2. Leader将Client的请求转化为Proposal，并分配Zxid
3. Leader为每个Follower分配一个单独的队列，然后将需要广播的Proposal依次放到队列中，并且根据FIFO策略进行消息发送
4. Follower接收到Proposal后，会首先将其以事务日志的方式写入本地磁盘中，写入成功后向 Leader反馈一个ACK响应消息
5. **Leader接收到超过半数以上Follower的ACK响应消息**后，即认为消息发送成 功,可以发送 commit消息
6. Leader向所有Follower广播commit消息，同时自身也会完成事务提交

## 数据模型

### Znode

<img src="{{ site.url }}/assets/img/2020-10-18-3.png" style="zoom:67%;" />

如图所示，ZooKeeper的内存数据模型由**树形的层级目录结构组成，其中的结点为Znode**，可以是文件也可以是目录。Znode分为持久结点和临时结点。**持久节点一旦被创建，不论Session情况一直存在，只有当client显式调用删除操作才会消失。临时结点会在客户端会话结束或者发生故障的时候被Zookeeper系统自动清除**。ZNode可以设置为自増属性(SEQUENTIAL)，自动将顺序编号赋予子结点名字

### Session

Session指client和服务器之间的TCP长连接。**通过这个连接，client能够通过心跳检测与服务器保持有效的会话，也能够向Zookeeper服务器发送请求并接受响应，同时还能够通过该连接接收来自服务器的Watch事件通知。** Session的`sessionTimeout`值用来设置一个客户端会话的超时时间。当由于服务器压力太大、网络故障或是客户端主动断开连接等各种原因导致客户端连接断开时，**只要在`sessionTimeout`规定的时间内能够重新连接上集群中任意一台服务器，那么之前创建的Session仍然有效。**在为client创建Session之前，服务端首先会为每个客户端都分配一个**全局唯一的Session id**。

### Watcher

Watcher事件监听器。**Zookeeper允许client在ZNode上注册一些Watcher，当一些特定事件触发的时候，ZooKeeper服务端会将事件通知到感兴趣的客户端上去**，该机制是Zookeeper实现分布式协调服务的重要特性。

### ACL

Zookeeper采用**ACL(*Access Control Lists*)策略来进行权限控制**，类似于 UNIX 文件系统的权限控制。Zookeeper 定义了如下5种权限

- CREATE: 创建子节点的权限
- READ: 获取节点数据和子节点列表的权限
- WRITE: 更新节点数据的权限
- DELETE: 删除子节点的权限
- ADMIN: 设置节点 ACL 的权限

### 版本

对应于每个ZNode，Zookeeper都会为其维护一个叫作 **Stat** 的数据结构，记录了以下数据版本

- version: 当前ZNode的版本
- cversion: 当前ZNode子节点的版本
- cversion: 当前ZNode的ACL版本

## 应用场景

应用Zookeeper的接口原语可以完成分布式环境下的协调工作

### Leader选举

分布式系统中一种经典的体系结构是主从结构，主控服务器负责全局管理控制工作，而从节点负责具体的任务计算或者数据存储管理工作。**领导者选举指为了防止单点失效，往往会采取一主一备或者一主多备，当主控服务器发生故障后，由某台备机接管主控服务器功能成为新的主控机。**

 Zookeeper在实现领导者选举时，将临时节点设置为Leader专用节点，存储Leader的地址信息及其他辅助信息。每个进程执行以下逻辑实现Leader选举过程

- 进程P读取临时结点内容并设置Watcher
  - 如果读取操作成功，说明目前已有Leader并获得Leader相关信息
  - 如果读取失败，说明目前无Leader，则进程P试图自己创建该临时节点，并将自己的相关信息写入成为Leader。Zookeeper会通知其他设置Watcher的进程
- 如果Leader发生故障，则临时结点会被清除，下次读取时触发Leader选举

### 配置管理

**Zookeeper可以用来进行配置信息的动态管理**。客户端通过在ZNode上存储配置文件并注册Watcher来及时收到配置文件的动态变化。

### 组成员管理

**组成员管理是动态监控一个组内成员的变化情况，便于主控服务器发现新加入的工作结点或者发现某个工作结点的故障。** 具体的可以通过临时结点来实现，如果创建这个节点的client会话结束或者是发生故障，ZooKeeper会自动清掉临时节点，同时如果有新节点加入也会创建新的临时结点。

### 任务分配

**任务分配指将不同的任务负载分别分配到多台可用服务器**。在Zookeeper中维护一个tasks结点和machines结点，监控进程通过在这两个结点下注册Watcher来完成任务的动态分配。通过machines结点下的子结点可以掌握整体的任务分配情况，通过tasks结点下的子结点可以监控新加入的任务。

### 分布式锁

如果要实现排他锁。对某个临界资源在Zookeeper上创建一个ZNode并设置为自增。**每当进程要获取临界资源时，在此ZNode下创建一个子结点，如果创建结点的编号是最小的那么就可以获得该资源，否则在比创建结点小1的结点上注册Watcher并等待。**

如果要实现读写锁。实现逻辑与排他锁类似，**写锁的获得条件判断为是否存在编号小于自己的其他锁，读锁的获得条件判断为是否存在编号小于自己的写锁。**

基于 Zookeeper 的 JUC 实现：[Menagerie](https://github.com/sfines/menagerie)

### 双向屏障同步

**屏障同步指多个并发进程都要到达某个同步点后再继续向后推进**，双向屏障同步指到达同步点后同时开始后续工作和同时最终结束。**每个进程在到达同步点后在某个结点下创建子结点，并判断子结点个数是否达到要求**，如果达到要求则可以开始后续工作。**如果进程完成工作，则删除自己创建的子结点并等待子结点个数为0进行结束动作**。

## REFERENCE

1. 大数据日知录
2. [可能是全网把 ZooKeeper 概念讲的最清楚的一篇文章——慕课网](https://zhuanlan.zhihu.com/p/44348274)

