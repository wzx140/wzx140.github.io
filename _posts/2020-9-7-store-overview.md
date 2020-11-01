---
layout: post
title:  "Spark源码阅读(六)：存储体系总览"
date:   2020-9-7
categories: Spark
keywords: Spark, 存储体系
mathjax: false
author: wzx
---

介绍Spark中的存储体系



## 整体介绍

<img src="{{ site.url }}/assets/img/2020-9-7-3.png" style="zoom: 67%;" />

- `BlockManagerMaster`: **代理`BlockManager`与Driver上的`BlockManagerMasterEndpoint`通信**。如图序号1和2所示，Driver和Executor上的`BlockManager`通过Driver上的`BlockManagerMasterEndpoint`通信。`BlockManagerMaster`之所以能够和`BlockManagerMasterEndpoint`通信，是因为它持有了`BlockManagerMasterEndpoint`的`RpcEndpointRef`

- `BlockManagerMasterEndpoint`: 由Driver上的`SparkEnv`负责创建和注册到Driver的`RpcEnv`中。**主要对各个节点上的`BlockManager`、`BlockManager`与Executor的映射关系及block位置信息（即block所在的`BlockManager`）等进行管理。**

- `BlockManagerSlaveEndpoint`: 每个Executor或Driver的`SparkEnv`中都有属于自己的`BlockManagerSlaveEndpoint`，分别由各自的`SparkEnv`负责创建和注册到各自的RpcEnv中。各自`BlockManager`的`slaveEndpoint`属性持有各自`BlockManagerSlaveEndpoint`的`RpcEndpointRef`。**如图序号3和4所示，用于接收`BlockManagerMasterEndpoint`的命令，如删除block、获取block状态、获取匹配的`BlockId`等**。

- `MemoryManager`: **内存管理器**。负责对单个节点上内存的分配与回收。


- `BlockTransferService`: **block传输服务**。**主要用于不同阶段的任务之间的lock数据的传输与读写**。如图序号5和6所示，Executor或者Driver上的`shuffleClient`通过远端的`BlockTransferService`提供的服务上传和下载Block。


- `DiskBlockManager`: **磁盘block管理器。对磁盘上的文件及目录的读写操作进行管理**。


- `BlockInfoManager`：**block信息管理器。负责对block的元数据及锁资源进行管理**。


- `MemoryStore`：**内存存储。依赖于`MemoryManager`，负责对block的内存存储**。


- `DiskStore`：**磁盘存储。依赖于`DiskBlockManager`，负责对block的磁盘存储**。

## REFERENCE

1. Spark内核设计的艺术：架构设计与实现
