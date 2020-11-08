---
layout: post
title: "Spark源码阅读(三十一): SparkSQL之物理计划"
date:   2020-11-8
categories: Spark
keywords: Spark, SparkSQL, 物理计划
mathjax: false
author: wzx
---

物理计划阶段，Spark SQL 根据逻辑算子树得到物理算子树。与逻辑计划的平台无关性不同，物理计划是与底层平台紧密相关的




## 物理计划流程

![]({{ site.url }}/assets/img/2020-11-8-1.png)

1. 由`SparkPlanner`将各种物理计划策略`Strategy`作用于对应的`LogicalPlan`节点上，生成`SparkPlan`列表。
2. 直接调用`next()`获取第一个计划
3. 提交前进行准备工作，进行一些分区排序方面的处理，确保`SparkPlan`各节点能够正确执行，这一步通过 `prepareForExecution()`方法调用若干规则进行转换。

## `SparkPlan`

在物理算子树中，**叶结点类型的`SparkPlan`结点负责创建RDD，每个分支结点类型的`SparkPlan`结点等价于在 RDD上进行一次Transformation，即通过调用`execute()`函数转换成新的RDD，最终执行`collect()`操作触发计算，返回结果给用户**。`SparkPlan`的继承体系如下图所示

![]({{ site.url }}/assets/img/2020-11-8-2.png)

### `LeafExecNode`

**没有子结点的物理执行计划**。如下图所示的继承关系`LeafExecNode`，负责对初始RDD的创建

![]({{ site.url }}/assets/img/2020-11-8-3.png)

- `RangeExec`: 利用`Spark­Context.parallelize()`方法生成给定范围的RDD
- `HiveTableScanExec`: 根据Hive数据表存储的HDFS信息直接生成`HadoopRDD`
- `FileSourceScanExec`: 根据数据表所在的源文件生成`FileScanRDD`

### `UnaryExecNode`

**只有一个子结点的物理执行计划**。`UnaryExecNode`结点的作用主要是对 RDD 进行转换操作

![]({{ site.url }}/assets/img/2020-11-8-4.png)

- `ProjectExec`: 对子节点产生的 RDD 进行列剪裁
- `FilterExec`: 分别对子节点产生的 RDD 进行 行过滤
- `Exchange`: 负责对数据进行重分区
- `SampleExec`: 对输入 RDD 中的数据进行采样
- `SortExec`: 按照一 定条件对输入 RDD 中数据进行排序
- `WholeStageCodegenExec`: 将生成的代码 整合成单个 Java 函数

### `BinaryExecNode`

**有两个子结点的物理执行计划**。除`CoGroupExec`外都是不同类型的 Join 执行计划。

![]({{ site.url }}/assets/img/2020-11-8-5.png)

`CoGroupExec`算子将两个要进行合并的左、右子`SparkPlan`所产生的 RDD ，按照相同的 key 值组合到一起，返回的左右子树key对应值的聚合结果

### 其他

![]({{ site.url }}/assets/img/2020-11-8-6.png)

## Metadata&Metrics

在某些`SparkPlan`的子类中使用`Map[String, String]`类型的变量`metadata`来存储元数据信息。一 般情况下，**元数据主要用于描述数据源的 一 些基本信息，例如数据文件的格式、存储路径等**。目前只有`FileSourceScanExec`和`RowDataSourceScanExec`两种叶结点物理计划对其进行了重载实现。

在`SparkPlan`的子类重载`Map[String, SQLMetric]`类型的变量`metrics`来存储指标信息。**在 Spark 执行过程中，`metrics`能够记录各种信息，为应用的诊断和优化提供基础。**

## 分区

**`SparkPlan.requiredChildDistribution()`规定了当前`SparkPlan`所需的数据分布列表**。在`SparkPlan`分区体系实现中，如下图所示，**特质`Partitioning`表示对数据进行分区的操作，特质 Distribution 则表示数据的分布**。`Partitioning.satisfies()`方法需要传入特质`Distribution`，判断`Partitioning`操作是否能满足所需的数据分布`Distribution`，即**是否需要对数据进行重分区**

![]({{ site.url }}/assets/img/2020-11-8-7.png)

### Distribution

定义了查询执行时，**同一个表达式下的不同数据元组在集群节点上的分布情况**。**节点间分区信息，即数据元组在集群不同的物理节点上是如何分区的**，可以用来判断聚合算子是否进行局部计算避免全局操作。**分区数据内排序信息**。有以下5种实现，实现了`requiredNumPartitions`属性和`createPartitioning()`方法

- `UnspecifiedDistribution`: **未指定分布，无需确定数据元组之间的位置关系**。调用`createPartitioning()`方法将抛出异常
- `AllTuples`: **只有一个分区，所有的数据元组存放在一起**。调用`createPartitioning()`方法将返回构造的`SinglePartition`
- `BroadcastDistribution`: **广播分布，数据会被广播到所有节点上**。构造参数为广播模式，原始数据`IdentityBroadcastMode`或`HashedRelation`对象`HashedRelationBroadcastMode`。调用`createPartitioning()`方法将返回构造的`BroadcastPartitioning`
- `ClusteredDistribution`: 传入一组表达式`Seq[Expression]`，**起到了哈希函数的效果**。调用`createPartitioning()`方法将返回构造的`HashPartitioning`，相同 value 的数据元组会被存放在一起
- `HashClusteredDistribution`: 传入一组表达式`Seq[Expression]`，**起到了哈希函数的效果**。调用`createPartitioning()`方法将返回构造的`HashPartitioning`，相同哈希值的数据元组会被存放在一起
- `OrderedDistribution`: 传入一组排序表达式`Seq[SortOrder]`，**数据元组会根据表达式计算后的结果排序**。调用`createPartitioning()`方法将返回构造的`RangePartitioning`

### Partitioning

**定义了一个物理算子输出数据的分区方式**。`satisfies()`方法会检查当前的 Partitioning 操作能否得到所需的数据分布`Distribution` 。 当不满足时，一般需要进行repartition()操作，对数据进行重新组织。如下图所示的继承关系

![]({{ site.url }}/assets/img/2020-11-8-8.png)

## Sort

**`SparkPlan.requiredChildOrdering() `规定了当前 SparkPlan所需的数据排序方式列表。**

- `outputOrdering`: `Seq[SortOrder]`类型，**指定每个partition内的顺序** 
- `requiredChildOrdering`: `Seq[Seq[SortOrder]]`类型，**指定每个子结点partition内的顺序** 

在`FileSourceScanExec`中，只有获取到bucket信息，才会构建`HashPartitioning`。只有获取到`sortColumns`(即数据源中用于排序的列)，才会构造`SortOrder`

```scala
override lazy val (outputPartitioning, outputOrdering): (Partitioning, Seq[SortOrder]) = {
  if (bucketedScan) {
    val spec = relation.bucketSpec.get
    val bucketColumns = spec.bucketColumnNames.flatMap(n => toAttribute(n))
    val partitioning = HashPartitioning(bucketColumns, spec.numBuckets)
    val sortColumns =
    spec.sortColumnNames.map(x => toAttribute(x)).takeWhile(x => x.isDefined).map(_.get)

    val sortOrder = if (sortColumns.nonEmpty) {
      val files = selectedPartitions.flatMap(partition => partition.files)
      val bucketToFilesGrouping =
      files.map(_.getPath.getName).groupBy(file => BucketingUtils.getBucketId(file))
      val singleFilePartitions = bucketToFilesGrouping.forall(p => p._2.length <= 1)

      if (singleFilePartitions) {
        sortColumns.map(attribute => SortOrder(attribute, Ascending))
      } else {
        Nil
      }
    } else {
      Nil
    }
    (partitioning, sortOrder)
  } else {
    (UnknownPartitioning(0), Nil)
  }
}
```

## Strategy

![]({{ site.url }}/assets/img/2020-11-8-9.png)

所有的策略都继承自`GenericStrategy`类，其中定义了**`planLater()`产生`PlanLater`包装逻辑计划为占位符，定义了`apply()`方法将传入的`LogicalPlan`转换为`SparkPlan`的列表，如果当前的执行策略无法应用于该 `LogicalPlan`节点，则返回空列表**。`SparkStrategy`继承自`GenericStrategy`类，实现了`planLater()`方法，各种具体的`Strategy`继承自`SparkStrategy`实现了`apply()`方法。

## Pattern

如下图所示，展示了Strategy的`apply()`方法中的**多对一的映射模式**。如果对应的`LogicalPlan`时，就会递归查找子节点，若子节点也是对应类型，则收集该子结点，直到碰到其他类型的`LogicalPlan`节点为止，最后生成映射的物理计划

- `ExtractEquiJoinKeys`: 针对具有相等条件的Join操作的算子集合，提取出其中的Join条件、左子节点和右子节点等信息 
- `ExtractFiltersAndinnerJoins`: 收集Inner类型Join操作中的过滤条件
- `PhysicalAggregation`: 针对聚合操作，提取出聚合算子中的各个部分，并对一些表达式进行初步的转换
- `PhysicalOperation`: 匹配逻辑算子树中的`Project`和`Filter`等节点，返回投影列、过滤条件集合和子节点



![]({{ site.url }}/assets/img/2020-11-8-10.png)

## 物理计划生成

<img src="{{ site.url }}/assets/img/2020-11-8-11.png" alt="image-20201020154942914" style="zoom:50%;" />

如图所示，`SparkPlanner`的继承关系。**生成物理计划的策略保存在`Strategies`属性下**。**`plan()`方法是生成物理计划的入口，将`strategies`依次应用到`LogicalPlan`，生成物理计划候选集合**。对于`PlanLater`类型的 `SparkPlan`，其`doExecute()`方法没有实现，表示不支持执行，所起到的作用仅仅是占位作用。`SparkPlanner`会取出所有的`PlanLater`算子，递归调用`plan()`方法进行替换。之后，调用`prunePlans()`方法对物理计划列表进行过滤，去掉一些不够高效的物理计划 。在生成`SparkPlan`列表后，调用`prepareForExecution()`根据需要插入shuffle操作和格式转换以适应Spark执行。

```scala
def plan(plan: LogicalPlan): Iterator[PhysicalPlan] = {
  // Obviously a lot to do here still...

  // Collect physical plan candidates.
  val candidates = strategies.iterator.flatMap(_(plan))

  // The candidates may contain placeholders marked as [[planLater]],
  // so try to replace them by their child plans.
  val plans = candidates.flatMap { candidate =>
    val placeholders = collectPlaceholders(candidate)

    if (placeholders.isEmpty) {
      // Take the candidate as is because it does not contain placeholders.
      Iterator(candidate)
    } else {
      // Plan the logical plan marked as [[planLater]] and replace the placeholders.
      placeholders.iterator.foldLeft(Iterator(candidate)) {
        // placeholder: PlanLater(logicalPlan)
        case (candidatesWithPlaceholders, (placeholder, logicalPlan)) =>
        // Plan the logical plan for the placeholder.
        val childPlans = this.plan(logicalPlan)

        candidatesWithPlaceholders.flatMap { candidateWithPlaceholders =>
          childPlans.map { childPlan =>
            // Replace the placeholder by the child plan
            candidateWithPlaceholders.transformUp {
              case p if p.eq(placeholder) => childPlan
            }
          }
        }
      }
    }
  }

  val pruned = prunePlans(plans)
  assert(pruned.hasNext, s"No plan for $plan")
  pruned
}
```

按照前一章所述的[逻辑优化过程]({% post_url 2020-11-5-spark-sql-optimizer-logical-plan %}#示例)，现在继续对`SELECT NAME FROM STUDENT WHERE AGE > 18 ORDER BY ID DESC`的物理计划生成过程做分析。如下所示是生成的物理计划

```
Project [NAME#11]
+- Sort [ID#10 DESC NULLS LAST], true, 0
   +- Project [NAME#11, ID#10]
      +- Filter (isnotnull(AGE#12) && (cast(AGE#12 as int) > 18))
         +- FileScan csv [ID#10,NAME#11,AGE#12] Batched: false, Format: CSV, Location: InMemoryFileIndex[file:/Users/wzx/Documents/tmp/spark_tmp/STUDENT.csv], PartitionFilters: [], PushedFilters: [IsNotNull(AGE)], ReadSchema: struct<ID:string,NAME:string,AGE:string>
```

## REFERENCE

1. Spark SQL内核剖析