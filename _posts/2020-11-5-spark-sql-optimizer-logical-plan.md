---
layout: post
title: "Spark源码阅读(三十): SparkSQL之优化逻辑计划"
date:   2020-11-5 9:00
categories: Spark
keywords: Spark, SparkSQL, 逻辑计划
mathjax: false
author: wzx
---

解析过的逻辑计划需要进行优化




## Optimizer

继承自`RuleExecutor`，并没有重写`execute()`方法，只是重写了`batches`属性，规定了用于优化的规则。其子类`SparkOptimizer`也是在其上增加了一些优化规则

![]({{ site.url }}/assets/img/2020-11-5-5.png)

## 优化规则

下面介绍一下常见的优化规则

### Finish Analysis

- `EliminateSubqueryAliases`: 消除`SubqueryAlias`结点。子查询仅用于提供查询的视角范围信息， 一旦解析阶段结束，该节点就可以被移除，**该优化规则直接将`SubqueryAlias`替换为其子节点**
- `EliminateView`: 删除视图
- `ReplaceExpressions`: 表达式替换，将`RuntimeReplaceable`的表达式替换为能够执行的正常表达式。**用来对其他类型的数据库提供兼容的能力**，例如可以用COALESCE来替换支持NVL的表达式
- `ComputeCurrentTime`: 计算与当前时间相关的表达式，为避免多个时间函数不一致，**`ComptuteCurrentTime`对逻辑算子树中的时间函数计算一次后，将其他同样的函数替换成该计算结果**
- `GetCurrentDatabase`: 获取当前数据库。**`GetCurrentDatabase`获得当前数据库， 然后用此结果替换所有的`CurrentDatabase`表达式**
- `RewriteDistinctAggregates`: 重写 Distinct 聚合操作，对于包含Distinct算子的聚合语句，将其转换为两个常规的聚合表达式，主要面向聚合查询

### Union

- `CombineUnions`: 用于将相邻的`Union`结点合并成一个`Union`结点，`flattenUnion()`方法**用栈实现了结点的合井**

  ```scala
  object CombineUnions extends Rule[LogicalPlan] {
    def apply(plan: LogicalPlan): LogicalPlan = plan transformDown {
      case u: Union => flattenUnion(u, false)
      // 嵌套Union中的distinct只需要保持最外存的distinct就可以了
      case Distinct(u: Union) => Distinct(flattenUnion(u, true))
    }
  
    private def flattenUnion(union: Union, flattenDistinct: Boolean): Union = {
      val stack = mutable.Stack[LogicalPlan](union)
      val flattened = mutable.ArrayBuffer.empty[LogicalPlan]
      // 如果不使用栈，需要使用递归方法去解决
      while (stack.nonEmpty) {
        stack.pop() match {
          case Distinct(Union(children)) if flattenDistinct =>
            stack.pushAll(children.reverse)
          case Union(children) =>
            stack.pushAll(children.reverse)
          case child =>
            flattened += child
        }
      }
      Union(flattened)
    }
  }
  ```

### Subquery

- `OptimizeSubqueries`: 在遇到`SubqueryExpression`表达式时， 进一步递归调用`Optimizer`对该表达式的子计划并进行优化

### Replace Operators

用来执行算子的替换操作。某些查询算子 可以直接改写为已有的算子，避免进行重复的逻辑转换

- `RewriteExceptAll`: **使用`Union`, `Aggregate`和`Generate`运算符的组合替换逻辑`Except`运算符**

  ```sql
  SELECT c1 FROM ut1 EXCEPT ALL SELECT c1 FROM ut2
  ==>
  SELECT c1
  FROM (
    SELECT replicate_rows(sum_val, c1)
      FROM (
        SELECT c1, sum_val
          FROM (
            SELECT c1, sum(vcol) AS sum_val
              FROM (
                SELECT 1L as vcol, c1 FROM ut1
                UNION ALL
                SELECT -1L as vcol, c1 FROM ut2
             ) AS union_all
           GROUP BY union_all.c1
         )
       WHERE sum_val > 0
      )
  )
  ```

- `ReplaceIntersectWithSemiJoin`：**将` Except`操作算子替换为`Left-Semi Join`操作算子**

  ```sql
  SELECT a1, a2 FROM Tab1 INTERSECT SELECT b1, b2 FROM Tab2
  ==>
  SELECT DISTINCT a1, a2 FROM Tab1 LEFT SEMI JOIN Tab2 ON a1<=>b1 AND a2<=>b2
  ```

- `ReplaceExceptWithFilter`: 如果逻辑`Except`运算符中的一个或两个数据集都是使用`Filter`进行转换的，则此规则将通过翻转右子对象的筛选条件，**将逻辑`Except`运算符替换为`Filter`运算符**

  ```sql
  SELECT a1, a2 FROM Tab1 WHERE a2 = 12 EXCEPT SELECT a1, a2 FROM Tab1 WHERE a1 = 5
  ==>
  SELECT DISTINCT a1, a2 FROM Tab1 WHERE a2 = 12 AND (a1 is null OR a1 <> 5)
  ```

- `ReplaceExceptWithAntiJoin`: **将`Except`操作算子替换为`left-anti Join`**

  ```sql
  SELECT a1, a2 FROM Tab1 EXCEPT SELECT b1, b2 FROM Tab2
  ==>  
  SELECT DISTINCT a1, a2 FROM Tab1 LEFT ANTI JOIN Tab2 ON a1<=>b1 AND a2<=>b2
  ```

- `ReplaceDistinctWithAggregate`: **将`Distinct`算子转换为`Aggregate`语句**

  ```sql
  SELECT DISTINCT f1, f2 FROM t 
  ==>
  SELECT f1, f2 FROM t GROUP BY f1, f2
  ```

### Aggregate

- `RemoveLiteralFromGroupExpressions`: 删除GROUPY BY中的常数
- `RemoveRepetitionFromGroupExpressions`: 删除GROUPY BY中的重复表达式

### Operator Optimizations

从整体来看，下表中的优化规则可以分为3个模块: **算子下推(Operator Push Down)、算子组合(Operator Combine)、常量折叠与长度削减(Constant Folding and Strength Reduction)**

- **算子下推**: 下表中前8条规则都属于算子下推的模块。算子下推所执行的优化操作主要是**将逻辑算子树中上层的算子节点尽量下推，使其靠近叶子节点，这样能够在不同程度上减少后续处理的数据量甚至简化后续的处理逻辑**。以列剪裁(`ColumnPruning`)为例，是查询语句中只涉及 A、 B 两列，那么会在读取数据后剪裁出这两列。 
- **算子组合**: 下表中从`CollapseRepartition`到`CombineUnions`都属于算子组合类型的优化。算子组合类型的优化规则**将逻辑算子树中相邻的能够进行组合的算子尽量整合在一起，避免多次计算以提高性能**
- **常量折叠与长度削减**: 下表后17条优化规则都属于常量折叠与长度削减的优化。**对于逻辑算子树中涉及某些常量的节点，可以在实际执行之前就完成静态处理**。例如`ConstantFolding`规则，对于能够折叠的表达式会直接在`EmptyRow`上执行`evaluate()`操作，从而构造新的`Literal`表达式

![]({{ site.url }}/assets/img/2020-11-5-6.png)

### LocalRelation

- `ConvertToLocaLRelation`: 将`LocalRelation`上的本地操作(不涉及数据交互)转换为另 一个 `LocaLRelation`，**转化操作`LocalRelation`的`Project`和`Limit`算子为`LocalRelation`**
- `PropagateEmptyRelation`: **折叠空的`LocalRelation`**

### User Provided Optimizers

用户自定义的优化规则

## 优化过程

按照前一章所述的[逻辑计划解析过程]({% post_url 2020-11-5-spark-sql-resolved-logical-plan %}#示例)，现在继续对`SELECT NAME FROM STUDENT WHERE AGE > 18 ORDER BY ID DESC`的analyzed逻辑计划优化过程做分析。仍然是从`Optimizer.execute()`方法为入口，开始是对逻辑计划树进行优化

- 应用`EliminateSubqueryAliases`优化规则，`SubqueryAlias`直接替换为子结点`Relation`
- 应用`InferFiltersFromConstraints`优化规则，对`Filter`结点的约束条件进行分析，添加额外的过滤条件列表`isnotnull(AGE#12)`

```
Project [NAME#11]
+- Sort [ID#10 DESC NULLS LAST], true
   +- Project [NAME#11, ID#10]
      +- Filter (isnotnull(AGE#12) && (cast(AGE#12 as int) > 18))
         +- Relation[ID#10,NAME#11,AGE#12] csv
```

## REFERENCE

1. Spark SQL内核剖析