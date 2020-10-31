---
layout: post
title:  "Spark源码阅读(八)：存储体系之磁盘存储"
date:   2020-9-8 9:00
categories: Spark
keywords: Spark SparkCore
mathjax: false
author: wzx
---

介绍Spark中block的磁盘存储





## `DiskBlockManager`

**为逻辑block与数据写入磁盘的物理文件位置之间建立逻辑的映射关系**

下面是成员属性

- `deleteFilesOnStop`: 停止`DiskBlockManager`的时候是否删除本地目录

- `subDirsPerLocalDir`:  磁盘存储`DiskStore`的本地子目录的数量，由`spark.diskStore.subDirectories`参数指定，默认64

- `localDirs`: **block落地本地文件根目录**，由`createLocalDirs()`方法获取

  - 由`Utils.getConfiguredLocalDirs()`获取到本地路径，默认获取`spark.local.dir`属性或者系统属性`java.io.tmpdir`指定的目录
  - 由`Utils.createDirectory()`方法在上一步获取到的各本地路径下新建以`blockmgr-UUID`为名的子目录

  ```scala
  private def createLocalDirs(conf: SparkConf): Array[File] = {
    Utils.getConfiguredLocalDirs(conf).flatMap { rootDir =>
      try {
        val localDir = Utils.createDirectory(rootDir, "blockmgr")
        logInfo(s"Created local directory at $localDir")
        Some(localDir)
      } catch {
        case e: IOException =>
        logError(s"Failed to create local dir in $rootDir. Ignoring this directory.", e)
        None
      }
    }
  }
  ```



- `subDirs`: 二维`File`数组，表示**本地目录和子目录名称的组合关系**，为每个`localDir`生成`subDirsPerLocalDir`个子目录。`private val subDirs = Array.fill(localDirs.length)(new Array[File](subDirsPerLocalDir))`。下图所示目录的层级结构

  <img src="{{ site.url }}/assets/img/2020-9-8-2.png" style="zoom:67%;" />

- `shutdownHook`: 为`DiskBlockManager`设置好关闭钩子

下面是成员方法

- `getFile()`: **由文件名或者`BlockId`获取文件**

  - 根据文件名的哈希值计算出`localDir`和`subDir`
  - 由`localDir`的`synchronized`进行同步，因为只有`subDir`才涉及到创建等线程不安全操作
  - 如果`subDir`没有则创建一个，返回一个在此二级目录下的文件

  ```scala
  def getFile(filename: String): File = {
    val hash = Utils.nonNegativeHash(filename)
    val dirId = hash % localDirs.length
    val subDirId = (hash / localDirs.length) % subDirsPerLocalDir

    val subDir = subDirs(dirId).synchronized {
      val old = subDirs(dirId)(subDirId)
      if (old != null) {
        old
      } else {
        val newDir = new File(localDirs(dirId), "%02x".format(subDirId))
        if (!newDir.exists() && !newDir.mkdir()) {
          throw new IOException(s"Failed to create local dir in $newDir.")
        }
        subDirs(dirId)(subDirId) = newDir
        newDir
      }
    }

    new File(subDir, filename)
  }

  def getFile(blockId: BlockId): File = getFile(blockId.name)
  ```

- `containsBlock)()`: `DiskBlockManager`中是否有这个block

- `getAllFiles()`: 返回 `DiskBlockManager`上所有的文件。**为了保证线程安全，采用同步克隆的方式，所以返回的文件列表可能不是最新的**

  ```scala
  def getAllFiles(): Seq[File] = {
    subDirs.flatMap { dir =>
      dir.synchronized {
        dir.clone()
      }
    }.filter(_ != null).flatMap { dir =>
      val files = dir.listFiles()
      if (files != null) files else Seq.empty
    }
  }
  ```

- `getAllBlocks()`: 调用了`getAllFiles()`方法，**获取所有储存在磁盘上的block**。可以看出block和文件是一一对应的

  ```scala
  def getAllBlocks(): Seq[BlockId] = {
    getAllFiles().flatMap { f =>
      try {
        Some(BlockId(f.getName))
      } catch {
        case _: UnrecognizedBlockId =>
        None
      }
    }
  }
  ```

- `createTempLocalBlock()`: 创建保存中间结果的block和文件。

- `createTempShuffleBlock()`: 创建保存中间shuffle结果的block和文件。

  ```scala
  def createTempLocalBlock(): (TempLocalBlockId, File) = {
    var blockId = new TempLocalBlockId(UUID.randomUUID())
    while (getFile(blockId).exists()) {
      blockId = new TempLocalBlockId(UUID.randomUUID())
    }
    (blockId, getFile(blockId))
  }

  def createTempShuffleBlock(): (TempShuffleBlockId, File) = {
    var blockId = new TempShuffleBlockId(UUID.randomUUID())
    while (getFile(blockId).exists()) {
      blockId = new TempShuffleBlockId(UUID.randomUUID())
    }
    (blockId, getFile(blockId))
  }
  ```

- `stop()`: 正常停止`DiskBlockManager`，调用了`doStop()`，内部调用了`Utils.deleteRecursively()`删除所有`localDirs`目录以及下面的文件

  ```scala
  private[spark] def stop() {
    try {
      ShutdownHookManager.removeShutdownHook(shutdownHook)
    } catch {
      case e: Exception =>
      logError(s"Exception while removing shutdown hook.", e)
    }
    doStop()
  }

  private def doStop(): Unit = {
    if (deleteFilesOnStop) {
      localDirs.foreach { localDir =>
        if (localDir.isDirectory() && localDir.exists()) {
          try {
            if (!ShutdownHookManager.hasRootAsShutdownDeleteDir(localDir)) {
              Utils.deleteRecursively(localDir)
            }
          } catch {
            case e: Exception =>
            logError(s"Exception while deleting local spark dir: $localDir", e)
          }
        }
      }
    }
  }
  ```

## `DiskStore`

**在磁盘上存储的block的底层管理器**。

有以下重要属性

- `minMemoryMapBytes`: 读取磁盘中的Block时，是直接读取还是使用FileChannel的[内存镜像映射方法](https://zhuanlan.zhihu.com/p/27698585)读取的阈值。由`spark.storage.memoryMapThreshold`指定，默认为2m。
- `maxMemoryMapBytes`
- `blockSizes`: `ConcurrentHashMap[BlockId, Long]`。**保存`BlockId`和block大小的映射关系**，保证线程安全
- `diskManager`: `DiskBlockManager`

有以下重要方法

- `getSize()`: 在`blockSizes`中根据`BlockId`取出block大小

- `contains()`: 由`DiskBlockManager.getFile()`获取文件对象并判断是否存在

- `put()`: 调用给定回调方法`writeFunc`将**对应的block写到磁盘**中，并在`blockSizes`中加入`BlockId`和大小的映射关系，流关闭失败时调用`remove()`方法

  ```scala
  def put(blockId: BlockId)(writeFunc: WritableByteChannel => Unit): Unit = {
    if (contains(blockId)) {
      throw new IllegalStateException(s"Block $blockId is already present in the disk store")
    }
    logDebug(s"Attempting to put block $blockId")
    val startTime = System.currentTimeMillis
    val file = diskManager.getFile(blockId)
    val out = new CountingWritableChannel(openForWrite(file))
    var threwException: Boolean = true
    try {
      writeFunc(out)
      blockSizes.put(blockId, out.getCount)
      threwException = false
    } finally {
      try {
        out.close()
      } catch {
        case ioe: IOException =>
        if (!threwException) {
          threwException = true
          throw ioe
        }
      } finally {
        if (threwException) {
          remove(blockId)
        }
      }
    }
    val finishTime = System.currentTimeMillis
    logDebug("Block %s stored as %s file on disk in %d ms".format(
      file.getName,
      Utils.bytesToString(file.length()),
      finishTime - startTime))
  }

  // 如果数据需要加密，则创建加密的channel，否则不加密
  private def openForWrite(file: File): WritableByteChannel = {
    val out = new FileOutputStream(file).getChannel()
    try {
      securityManager.getIOEncryptionKey().map { key =>
        CryptoStreamUtils.createWritableChannel(out, conf, key)
      }.getOrElse(out)
    } catch {
      case e: Exception =>
      Closeables.close(out, true)
      file.delete()
      throw e
    }
  }
  ```

- `putBytes()`: **将`ByteBuffer`写入到磁盘中的一个block中**，调用`put()`方法

  ```scala
  def putBytes(blockId: BlockId, bytes: ChunkedByteBuffer): Unit = {
    put(blockId) { channel =>
      bytes.writeFully(channel)
    }
  }
  ```

- `getBytes()`: 由`DiskBlockManager`**获取对应block**的`File`实例，并封装成加密的`EncryptedBlockData`或者`DiskBlockData`返回。

  ```scala
  def getBytes(blockId: BlockId): BlockData = {
    val file = diskManager.getFile(blockId.name)
    val blockSize = getSize(blockId)

    securityManager.getIOEncryptionKey() match {
      case Some(key) =>
      new EncryptedBlockData(file, blockSize, conf, key)

      case _ =>
      new DiskBlockData(minMemoryMapBytes, maxMemoryMapBytes, file, blockSize)
    }
  }
  ```

- `remove()`: 由`DiskBlockManager`**删除block文件和`blockSize`上的映射关系**

  ```scala
  def remove(blockId: BlockId): Boolean = {
    blockSizes.remove(blockId)
    val file = diskManager.getFile(blockId.name)
    if (file.exists()) {
      val ret = file.delete()
      if (!ret) {
        logWarning(s"Error deleting ${file.getPath()}")
      }
      ret
    } else {
      false
    }
  }
  ```


## REFERENCE

1. Spark内核设计的艺术：架构设计与实现

