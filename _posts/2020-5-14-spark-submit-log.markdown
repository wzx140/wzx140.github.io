---
layout: post
title:  "在spark-submit中自定义日志配置"
date:   2020-5-14
categories: Spark
tags: Distribution Spark
mathjax: true
author: wzx
---

- 目录
{:toc}

日志有助于debug和优化程序，对于spark程序而言，有时需要改变默认的日志配置，如调整日志输出级别，根据日志级别进行分流存储等




## 调整日志级别
在代码中直接指定日志记录级别即可
```scala
// 通过sparkConetext指定
spark = SparkSession.builder.getOrCreate()
spark.sparkContext.setLogLevel("ERROR")

// 通过logger指定
import org.apache.log4j.{Level, Logger}
val rootLogger = Logger.getRootLogger()
rootLogger.setLevel(Level.ERROR)
```

以上的方法只能修改日志输出级别，并且写死在代码中，一旦需要修改还要重新编译，建议采用配置文件的形式。
```properties
log4j.rootLogger=info,console,logFile
log4j.additivity.org.apache=true
# 控制台(console)
log4j.appender.console=org.apache.log4j.ConsoleAppender
log4j.appender.console.Threshold=error
log4j.appender.console.Target=System.err
log4j.appender.console.layout=org.apache.log4j.PatternLayout
log4j.appender.console.layout.ConversionPattern=[%-5p] %d(%r) --> [%t] %l: %m %x %n
```


## 使用配置文件
如果只是在idea上执行，只需**将`log4j.properties`放在`resources`目录下**即可。因为`resources`下的资源文件在编译时会自动放入classpath目录下，idea是以`java mainClass`的命令将spark程序作为java应用运行的，所以程序可以加载到`log4j.properties`配置文件。如果以`spark-submit`命令提交，放在**jar包里的`log4j.properties`就会失效**。

### 本地提交
如果只是在本地提交运行，需要手动为driver和executor指定`-Dlog4j.configuration=file:path_of_file`，如果是一个文件，**`file:`前缀必须要加上**，文件地址可以是相对地址也可以是绝对地址

```shell
spark-submit \
    --master "local[2]" \
    --conf "spark.driver.extraJavaOptions=-Dlog4j.configuration=file:path_of_file" \
    --conf "spark.executor.extraJavaOptions=-Dlog4j.configuration=file:path_of_file" \
    --class your_main_class \
    "your jar path"
```

## 集群提交
如果需要提交到集群上运行，就需要考虑到**driver和executor运行在不同的物理机上**。如果结点较少，可以将配置文件手动上传到所有结点上，在`spark.driver.extraJavaOptions`和`spark.executor.extraJavaOptions`分别指定对应路径即可。下面介绍通过`--files`选项指定配置文件与应用一起提交。这里以提交到yarn上运行作为示例。

### client
在Yarn-client模式下，driver在本地运行，所以**driver需要指定本地配置文件地址**

```shell
spark-submit \
    --master yarn \
    --deploy-mode client \
    --conf "spark.driver.extraJavaOptions=-Dlog4j.configuration=file:path_on_client" \
    --conf "spark.executor.extraJavaOptions=-Dlog4j.configuration=file:log4j.properties" \
    --files "/absolute/path/to/your/log4j.properties" \
    --class your_main_class \
    "your jar path"
```

### cluster
在Yarn-cluster模式下，driver在集群上运行，所以**`--files`指定的文件也提交到driver上**。

```shell
spark-submit \
    --master yarn \
    --deploy-mode cluster \
    --conf "spark.driver.extraJavaOptions=-Dlog4j.configuration=file:log4j.properties" \
    --conf "spark.executor.extraJavaOptions=-Dlog4j.configuration=file:log4j.properties" \
    --files "/absolute/path/to/your/log4j.properties" \
    --class your_main_class \
    "your jar path"
```

```
# client端提交日志
20/05/14 23:03:29 INFO yarn.Client: Uploading resource file:/home/wzx/package/log4j-ex.properties -> hdfs://master:8020/user/wzx/.sparkStaging/application_1589462736793_0003/log4j-ex.properties

# executor端日志
YARN executor launch context:
  env:
    CLASSPATH -> {{PWD}}<CPS>{{PWD}}/__spark_conf__<CPS>{{PWD}}/__spark_libs__/*<CPS>$HADOOP_CONF_DIR<CPS>$HADOOP_COMMON_HOME/share/hadoop/common/*<CPS>$HADOOP_COMMON_HOME/share/hadoop/common/lib/*<CPS>$HADOOP_HDFS_HOME/share/hadoop/hdfs/*<CPS>$HADOOP_HDFS_HOME/share/hadoop/hdfs/lib/*<CPS>$HADOOP_YARN_HOME/share/hadoop/yarn/*<CPS>$HADOOP_YARN_HOME/share/hadoop/yarn/lib/*<CPS>$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/*<CPS>$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/lib/*<CPS>{{PWD}}/__spark_conf__/__hadoop_conf__
    SPARK_YARN_STAGING_DIR -> hdfs://master:8020/user/wzx/.sparkStaging/application_1589462736793_0004
    SPARK_USER -> wzx

  command:
    {{JAVA_HOME}}/bin/java \
      -server \
      -Xmx1024m \
      '-Dlog4j.configuration=file:log4j-ex.properties' \
      -Djava.io.tmpdir={{PWD}}/tmp \
      '-Dspark.driver.port=40031' \
      '-Dspark.ui.port=0' \
      -Dspark.yarn.app.container.log.dir=<LOG_DIR> \
      -XX:OnOutOfMemoryError='kill %p' \
      org.apache.spark.executor.CoarseGrainedExecutorBackend \
      --driver-url \
      spark://CoarseGrainedScheduler@slave1:40031 \
      --executor-id \
      <executorId> \
      --hostname \
      <hostname> \
      --cores \
      1 \
      --app-id \
      application_1589462736793_0004 \
      --user-class-path \
      file:$PWD/__app__.jar \
      1><LOG_DIR>/stdout \
      2><LOG_DIR>/stderr

  resources:
    __app__.jar -> resource { scheme: "hdfs" host: "master" port: 8020 file: "/user/wzx/.sparkStaging/application_1589462736793_0004/spark-log-1.0-SNAPSHOT-jar-with-dependencies.jar" } size: 538108 timestamp: 1589468834449 type: FILE visibility: PRIVATE
    __spark_libs__ -> resource { scheme: "hdfs" host: "master" port: 8020 file: "/user/wzx/.sparkStaging/application_1589462736793_0004/__spark_libs__1732228556517610975.zip" } size: 305542283 timestamp: 1589468834061 type: ARCHIVE visibility: PRIVATE
    __spark_conf__ -> resource { scheme: "hdfs" host: "master" port: 8020 file: "/user/wzx/.sparkStaging/application_1589462736793_0004/__spark_conf__.zip" } size: 221578 timestamp: 1589468835273 type: ARCHIVE visibility: PRIVATE
    log4j-ex.properties -> resource { scheme: "hdfs" host: "master" port: 8020 file: "/user/wzx/.sparkStaging/application_1589462736793_0004/log4j-ex.properties" } size: 816 timestamp: 1589468834928 type: FILE visibility: PRIVATE
```
由日志可以看出，实际上本地配置文件被发送到了hdfs之上，executor在执行时会从这个路径下读取配置文件

### 在代码中指定

有时候，我们不能够使用 `--file`选项，也不能上传文件到每个机器上，这样读取配置文件就比较麻烦。但是Spark集群必须可以访问到你有权限的文件系统，以下以HDFS为例。直接指定`-Dlog4j.configuration=hdfs://hdfs_cluster/path/log4j-ex.properties`是无效的。可以通过`PropertyConfigurator.configure()`在代码中主动加载配置文件，由于是在集群环境下，所有driver和所有的executor都需要执行这个语句

```scala
val prop = new Properties()
prop.load(fs.open(new Path(FilePath.LOG_CONF_FILE)))
// 在driver中
PropertyConfigurator.configure(prop)
// 为空的partition不会执行语句
val partition = sc.defaultParallelism
sc.parallelize(0 to partition).repartition(partition)
.foreachPartition { _ =>
  // 在executor中加载
  PropertyConfigurator.configure(prop)
}
```



## REFERENCE
[1] [spark doc: Debugging your Application](https://spark.apache.org/docs/latest/running-on-yarn.html#debugging-your-application)  
[2] [stackoverflow: how to stop info messages displaying on spark console](https://stackoverflow.com/a/43747948/10569558)