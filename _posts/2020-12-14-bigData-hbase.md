---
dddddlayout: post
title:  "大数据日知录学习(八)：HBase与BigTable"
date:   2020-12-14
categories: 大数据理论与架构
keywords: HBase BigTable
mathjax: false
author: wzx
---

BigTable是一种针对海量结构化数据的分布式列式存储的数据库。HBase是参考BigTable论文的开源实现。




## 数据模型

BigTable是一个稀疏的，分布式的，持久化的三维有序Map，**最基础的存储单元是由(行键, 列主键, 时间)所定位的，其中列主键由列族和列描述符组成**。所以BigTable可以随时对表格的列进行增删的，而且每行只存储列内容不为空的数据。

- **行键(RowKey)**: 可以由任意的字符串组成，最大长度为64K，在内部存储中会被存储为字节数组，**表中的数据是按照行键的字典序排列的**
- **列族(Column Family)**: **列族是由一系列的列组成的，必须在创建表的时候就指定好**。列族不仅能帮助开发者构建数据的语义边界，还能有助于开发者设置某些特性，例如可以指定某个列族内的数据压缩形式。
- **列(Column)**: 一般从属于某个列族，一个列族中可以有数百万列且这些列都可以动态添加
- **时间戳(Timestamp)**: 同一信息随着时间变化的不同版本
- **单元格(Cell)**: `(row:string, column:string, time:int64) -> string`，**单元格中的数据是没有类型的，全部都是字节序列**

由于BigTable是面向海量数据存储的，所以在实际存储时，**根据行键进行切割，将一个Table切割为多个行键连续的行组成的Tablet**，由Tablet Server来管理每个Tablet。

## 整体架构
![]({{ site.url }}/assets/img/2020-12-14-1.png)

- Master Server: 负责整个系统的管理工作，包括**Tablet的分配、Tablet Server的负载均衡、Tablet Server失效检测**等
- Tablet Server: **负责Tablet数据的存储和管理**，需要响应客户端程序的读写请求，**Tablet以多个SSTable文件存储在GFS中**
- Client: 具体应用的接口程序，**直接和Tablet Server进行交互通信**，来读写某个子表对应的数据

## 元数据管理

BigTable中的元数据管理依赖Chubby。Chubby中有一个Servers目录，**每个Tablet Server在该目录下生成对应的文件，记载了这个Tablet Server的IP地址等管理信息**。

![]({{ site.url }}/assets/img/2020-12-14-2.png)

如图所示，**Tablet的寻址信息以MetaData Table的形式在BigTable集群中保存，类似B+树的结构**。Chubby文件保存了这棵树的根结点即第一个MetaData Table，是一个不可分割的Root Tablet。

- MetaData Tablet: **用于保存其他Tablet地址信息的特殊Tablet**。每一行以表名和Tablet最后一个行键的编码作为这一行的行键，在单元格中存储了Tablet的位置信息
  - Root tablet: 不可分割的tablet，**记录下一层MetaData Table的位置**，相当于树根结点
  - Other MetaData tablet: **具体记录用户Tablet位置信息的tablet**，相当于数的分支结点

## Master Server

Master Server专门负责管理工作，比如**处理新的Tablet Server加入，处理Tablet Server因为各种故障原因不能提供服务的容错问题，处理Tablet Server之间的负载均衡等**。

![]({{ site.url }}/assets/img/2020-12-14-3.png)

如图所示，为**Master Server启动时的运行流程**，首先在Chubby中获得一个 Master锁，这样可以阻止其他Master Server再次启动

1. 读取Chubby的Servers目录，从该目录下的文件可以**获得每个Tablet Server的地址信息**
2. 与Tablet Server通信，**获知每个Tablet Server存储了哪些Tablet并记录在内存管理数据中**
3. 读取Chubby的root节点可以**获取MetaData Table中的元数据**，这里记录了所有Tablet的信息
4. 将Metadata和Tablet Server反馈的信息进行对比，如果一部分Tablet在Metadata中，但是没有Tablet Server负责存储，说明这些Tablet是未获得分配的内容，所以**将这些Tablet信息加入末分配Tablet集合中，在以后分配给负载较轻的Tablet Server**

**当有新的Tablet Server加入时**，这个Tablet Server会在 Chubby 的 Servers 目录下生成对应的文件，Master Server通过周期性地扫描 Servers目录下文件可以很快获知有新Tablet Server的加入，之后可以将高负载的其他Tablet Server的部分数据,或者是未分配Tablet 中的数据交由新加入的服务器来负责管理

Master Server会周期性地询问Tablet Server的状态，**当Tablet Server宕机后**，会将 Chubby的 Servers目录下对应的文件删除、并将这个Tablet Server负责管理的Tablet放入末未分配Tablet中，在以后分配给负载较轻的Tablet Server

## SSTable&MemTable

SSTable和MemTable的存储结构是个典型的[LSM树]({% post_url 2020-9-5-bigData-data-structure %}#lsm)结构

![]({{ site.url }}/assets/img/2020-12-14-4.png)

**SSTable是在磁盘上用来存储Tablet数据的文件，如果加载进内存就是MemTable**。如图所示，每个 SSTable划分为两块

- **数据存储区**: 存储具体的数据，本身又被划分成小的数据块，每次读取的单位就是一个数据块
- **索引区**: 记录每个数据块存储的行键范围及其在SSTable中的位置信息

当 BigTable打开一个 SSTable文件的时候，系统将索引区加载入内存。当要读取一个数据块时，首先在内存的数据块索引中利用**二分査找**快速定位到数据块中的位置，之后就可以根据位置信息一次性读取某个数据块。

## Tablet Server

Tablet Server主要完成以下功能

- 存储管理子表数据,包括子表存储、子表恢复、子表分裂、子表合并等
- 响应客户端程序对子表的写请求
- 响应客户端程序对子表的读请求

### IO

![]({{ site.url }}/assets/img/2020-12-14-5.png)

如图所示，**对Tablet内容的更新包括插入或删除行数据、或者插入删除某行的某个列数据等操作**

1. 当Tablet Server接收到数据更新新请求时，首先将更新命令记入 CommitLog文件中
2. 之后将更新数据写入内存的MemTable结构中
3.  当MemTable里容纳的数据超过设定大小时，作为SSTable保存到GFS

如图所示，当**从Tablet Server读取数据**时

1. 使用[布隆过滤器]({% post_url 2020-9-5-bigData-data-structure %}#bloom-filter)快速判断某个SSTable文件是否包含要读取数据的行键
2. 在内存中的MemTable中查找数据
3. 如果没有查找内存中的SSTable索引文件并从SSTable数据文件中读取具体数据

### Compaction

![]({{ site.url }}/assets/img/2020-12-14-6.png)

如图所示，共有3种合并方式。在合并过程中，会将已经标记为删除的记录抛弃，有效回收存储资源

- **Minor Compaction**: 当Memtable写入数据过多时,会将内存中的Memtable写入磁盘中一个新的 SSTable中
- **Merging Compaction**: MemTable和部分SSTable合并
- **Major Compaction**: MemTable和所有SSTable合并

### Recovery

![]({{ site.url }}/assets/img/2020-12-14-7.png)

如图所示，当Tablet Server失败重启后，执行以下Tablet恢复机制

1. 从MetaData Table中读取管理信息，包括Tablet Server负责管理的Tablet对应的SSTable文件，以及 CommitLog对应的恢复点(Redo Point)
2. 根据CommitLog恢复点，恢复从这个位置之后的所有更新行为到Memtable中，完成了Memtable的重建工作
3. 将SSTable索引文件读入内存

### Split

Tablet Server负责Tablet分裂的管理，当某个Tablet存储的数据量过大，会将其分裂为两个均等大小的Tablet，并将相应的管理信息传递到MetaData中记录。

## HBase

在技术栈上，BigTable使用的GFS与Chubby对应HBase使用的**HDFS和ZooKeeper**。Bigtable的列族更像是一个逻辑概念，多个列族可以组成一个Locality Group。同一个Locality Group中的多个列族数据可以被存储在同一个SSTable文件中。**HBase中的不同的列族的数据是隔离在不同的存储路径中的**。**HBase中只有Minor Compaction和Major Compaction**。

![]({{ site.url }}/assets/img/2020-12-14-8.png)

如上图所示，为HBase的主要架构

- `Client`：包含访问HBase的接口
- `Zookeeper`：通过选举保证任何时候集群中只有一个HMaster。HMaster与Region Server启动时向其注册、存储所有Region的寻址入口、实时监控Region Server的上下线信息并实时通知给HMaster。
- `HMaster`：为Region Server分配Region、负责Region Server的负载均衡、发现失效的Region Server并重新分配其上的Region、管理用户对Table的增删改查
- `Region Server`：维护Region并处理对Region的IO请求、切分在运行过程中变得过大的Region
  - `Region`：HBase自动把表**水平划分**成region。每个区域由表中行的子集构成。**每个区域由它所属于的表、它所包含的第一行及其最后一行（不包括这行）来表示**。一开始，一个表只有一个region。等到region超出设定的大小阈值，分裂成两个大小基本新region。**region是在HBase 集群上分布数据的最小单位**。
    - `store`：数据块(列族)
      - `MemStore`：内存存储，缓存
      - `HFile`：文件存储
  - `HLog`：预写日志WAL，每一个业务数据的写入操作（PUT / DELETE）执行前，都会记账在WAL中

## REFERENCE

1. 大数据日知录
2. Chang F, Dean J, Ghemawat S, et al. Bigtable: A distributed storage system for structured data[J]. ACM Transactions on Computer Systems, 2008, 26(2): 4.

