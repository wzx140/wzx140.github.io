---
layout: post
title:  "Spark源码阅读(二十八): SparkSQL之unresolved逻辑计划"
date:   2020-10-21
categories: Spark
keywords: Spark, SparkSQL, 逻辑计划
mathjax: false
author: wzx
---

SQL语句经由ANTLR4解析树转换为未解析的逻辑算子树，逻辑计划本质上是一种中间过程表示，与Spark平台无关， 后续阶段会进一步解析占位符并映射为可执行的物理计划




## ConstraintHelper

特质`ConstraintHelper`是用于**推断约束规则**的，约束规则本质上属于数据过滤条件的一种，同样是`Expression`类型。有以下方法

- `inferAdditionalConstraints()`: **从给定的相等约束集合中推断出一组附加约束**。例如`a=5, a=b -> b=5`
- `constructIsNotNullConstraints()`: **从非空表达式以及可能空的属性推断出一组`isNotNull`约束**。例如`a>5 -> isNotNull(a)`

## QueryPlanConstraints

特质`QueryPlanConstraints`继承了`ConstraintHelper`。有以下方法

- `validConstraints`: 返回此结点的合法约束，由继承的子类实现

- `constraints`: 将`validConstraints`经由父类的`inferAdditionalConstraints()`和`constructIsNotNullConstraints()`方法进行约束推断后得到新的`ExpressionSet`约束

  ```scala
  if (conf.constraintPropagationEnabled) {
    ExpressionSet(
      validConstraints
      .union(inferAdditionalConstraints(validConstraints))
      .union(constructIsNotNullConstraints(validConstraints, output))
      .filter { c =>
        c.references.nonEmpty && c.references.subsetOf(outputSet) && c.deterministic
      }
    )
  } else {
    ExpressionSet(Set.empty)
  }
  ```

## AnalysisHelper

`AnalysisHelper`定义**与query analysis相关的方法**，通过`analyzed`属性来标识该计划是否被分析，防止重复分析。有以下重要的方法

- `resolveOperatorsUp()`: **将偏函数`PartialFunction[LogicalPlan, LogicalPlan]`规则递归应用于树并返回此结点的副本**，当规则不适用或者子树已经标记为已分析，则保持不变。**后序遍历，先对子结点应用规则再对当前结点应用规则**
- `resolveOperatorsDown()`: 作用与`resolveOperatorsUp()`相同。**先序遍历，先对当前结点应用规则再对子结点应用规则**
- `resolveOperators()`: 实际上调用了`resolveOperatorsDown()`
- `resolveExpressions()`: 规则类型为`PartialFunction[Expression, Expression]`，实际调用了`resolveOperatorsUp()`**只对`Expression`类型的结点应用规则**

## QueryPlan

**抽象计划类，其子类为逻辑计划和物理计划**。`QueryPlan`的主要操作分为5个模块，分别是输入输出、 字符串、 规范化、表达式操作、基本属性，如下图所示

<img src="{{ site.url }}/assets/img/2020-10-21-1.png" style="zoom:67%;" />

- 基本属性

  - `schema`: 当前结点的输出`StructType`类型，将`output()`的结果封装成`StructType`

  - `allAttributes`: 结点所涉及的所有属性列表`AttributeSeq`，返回所有子结点的`output()`

  - `references`: 结点表达式中所涉及的所有属性集合`AttributeSet`

    ```scala
    def references: AttributeSet = AttributeSet(expressions.flatMap(_.references))
    ```

  - `subqueries()`, `innerChildren()`: 结点包含的所有子查询`PlanExpression`

- 输入输出

  - `output()`: 抽象方法，返回`Seq[Attribute]`

  - `outputSet()`: 把`output()`返回的结果封装成`AttributeSet`

  - `inputSet()`: 获取当前结点的输入属性集`AttributeSet`，将所有子结点的输出属性封装成`AttributeSet`返回

    ```scala
    def inputSet: AttributeSet =
    	AttributeSet(children.flatMap(_.asInstanceOf[QueryPlan[PlanType]].output))
    ```

  - `producedAttributes()`: 该结点产生的属性集`AttributeSet`，当前返回空属性集由子类实现

  - `missingInput()`: 该结点表达式中涉及的但是其子结点和当前结点输出中并不包含的属性

    ```scala
    def missingInput: AttributeSet = references -- inputSet -- producedAttributes
    ```

- 字符串: 用于打印树型结构信息的方法。其中`statePrefix()`方法用来表示节点对应计划状态的前缀字符串

- 规范化:  `canonicalized()`对当前结点及其子结点进行规范化，`smaeResult()`则比较两个`QueryPlan`规范后是否相等

- 表达式操作

  - `transformExpressionsDown()`: 对该结点表达式遍历，并对每个结点调用了父类方法`TreeNode.transformDown()`并应用偏函数规则`PartialFunction[Expression, Expression]`
  - `transformExpressionsUp()`: 对该结点表达式遍历，并对每个结点调用了父类方法`TreeNode.transformUp()`并应用偏函数规则`PartialFunction[Expression, Expression]`
  - `transformExpressions`: 调用了`transformExpressionsDown()`
  - `transformAllExpressions()`: 对此结点及其所有子结点上调用`transformExpressions()`
  - `expressions()`: 返回此结点中的所有表达式

## LogicalPlan

如下图所示，`LogicalPlan`继承了`QueryPlan`和`QueryPlanConstraints`和`AnalysisHelper`，作为数据结构**记录了对应逻辑算子树节点的基本信息和基本操作**。

![]({{ site.url }}/assets/img/2020-10-21-2.png)

- `isStreaming`: 此逻辑算子树是否含有流式数据源
- `maxRows`: 此逻辑计划最大可能计算的行数
- `maxRowsPerPartition`: 此逻辑计划的每个partition最大可能计算的行数
- `resolved`, `childrenResolved`: 当前表达式和子结点时候含有未解析的占位符

有以下与解析相关的方法，占位符字符串的格式为`[scope].AttributeName.[nested].[fields]...`

- `resolve()`: **根据此逻辑计划的输出将给定的字符串或者`StructType`(占位符)解析为`NamedExpression`表达式**
- `resolveQuoted()`: 给定属性名根据符号点进行分割，并根据此逻辑计划的输出解析成`NamedExpression`
- `resolveChildren`: 根据此逻辑计划所有子节点的输入将给定的字符串解析为`NamedExpression`

### LeafNode

**没有子结点的逻辑计划**，重载了`children`返回空序列。由下图所示的继承关系可以看出，叶子结点一般代表着数据源，如RDD, hive, 集合等

![]({{ site.url }}/assets/img/2020-10-21-4.png)

## Command

**代表着系统执行的非查询命令的逻辑结点**，比如说DDL操作，相比于查询，执行的优先级更高。有以下继承关系，其方法`run()`实现了与命令相关的逻辑

- 特质`DataWritingCommand`: 用于写出数据或者更新指标的命令

  - 样例类`CreateDataSourceTableAsSelectCommand`: 用于使用查询结果创建数据源表的命令，如下所示

    ```sql
    CREATE TABLE [IF NOT EXISTS] [db_name.]table_name
       USING format OPTIONS ([option1_name "option1_value", option2_name "option2_value", ...])
       AS SELECT ...
    ```

  - 样例类`CreateHiveTableAsSelectCommand`: 建表并插入查询结果

  - 样例类`InsertIntoHadoopFsRelationCommand`: 将数据写入HDFS

  - 特质`SaveAsHiveFile`

    - 样例类`InsertIntoHiveDirCommand`: 将查询结果写入到文件系统中的hive语句
    - 样例类`InsertIntoHiveTable`: 将数据写入hive表

- 特质`RunnableCommand`: 直接运行的命令。有许多实现此特质的样例类


### BinaryNode

**有两个子结点的逻辑计划结点**，有以下继承关系

- 样例类`CoGroup`: full outer join
- 样例类`Join`
- 样例类`OrderedJoin`: 实现的`output()`方法返回左右子结点的输出合并
- 抽象类`SetOperation`: 集合操作
  - 样例类`Except`: 实现的`output()`方法只返回左子结点的输出，`validConstraints()`也只返回左子结点的约束
  - 样例类`Intersect`: 实现的`output()`方法只返回左子结点的输出，当左右子结点中有一个为可空时返回可空的，`validConstraints()`也返回左右子结点的约束合并

### UnaryNode

**只有一个子结点的逻辑计划结点**，常用于对数据的逻辑转换操作，下面介绍一些常见的样例类

- `Project`: 表示 SELECT 语句中选中列的那部分。包含了选中列的表达式`NamedExpression`

  ```scala
  case class Project(projectList: Seq[NamedExpression], child: LogicalPlan)
  	extends OrderPreservingUnaryNode
  ```

- `Filter`: 表示 WHERE 语句中的条件。包含了布尔表达式`Expression`

  ```scala
  case class Filter(condition: Expression, child: LogicalPlan)
  	extends OrderPreservingUnaryNode with PredicateHelper
  ```

- `Sort`: 表示 ORDER BY(全局排序)和SORT BY(分区排序)

  ```scala
  case class Sort(order: Seq[SortOrder], // 排序的字段或者表达式，还有排序方向
                  global: Boolean, // 否为全局的排序，还是分区的排序
                  child: LogicalPlan) extends UnaryNode
  ```

- `Distinct`: 表示SELECT中带有DISTINCT关键字的列

  ```scala
  case class Distinct(child: LogicalPlan) extends UnaryNode
  ```

- `Aggregate`: 

  ```scala
  case class Aggregate(
      groupingExpressions: Seq[Expression],  // GROUP BY 的字段
      aggregateExpressions: Seq[NamedExpression],   // SELECT 的字段
      child: LogicalPlan)
    extends UnaryNode
  ```

### 其他子类

- `InsertIntoTable`: 向表中插入数据，这个结点是未解析的，在分析阶段会被替换
- `CreateTable`: 创建一个表，可选插入数据，这个结点是未解析，在分析阶段会被替换
- `View`: 保存视图的元数据和输出，分析阶段结束后被删除
- `Union`: 组合多个逻辑计划，相当于UNION ALL
- `AppendData`: 在已存在的表中追加数据
- `ObjectProducer`: 用于产生只包含object列的行数据

## Unresolved逻辑计划树生成

<img src="{{ site.url }}/assets/img/2020-10-21-3.png" alt="image-20201026152250721" style="zoom:67%;" />

### AstBuilder&AbstractSqlParser

`SqlBaseBaseVistor`为ANTLR生成的类用于遍历ANTLR4解析树，**`AstBuilder`继承了`SqlBaseBaseVistor`并实现了其中的一部分访问者方法，将ANTLR4解析树结点转换为逻辑计划**。以`visitSingleStatement()`方法为例

- `withOrigin()`方法根据ANTLR4解析树结点上下文`ctx`注册当前sql语句的行和开始字符处
- 方法内部先调用了`visit()`方法，如下所示，触发visitor遍历模式，根据ANTLR4解析树结点的不同调用不同`visitXXX()`方法
- 子结点都转换完，才会生成当前结点的逻辑计划

```scala
// AstBuilder
override def visitSingleStatement(ctx: SingleStatementContext): LogicalPlan = withOrigin(ctx) {
  // 这里是递归
  // ctx.statement里调用了最终调用了ParserRuleContext.getChild(type, 0)返回第一个子结点
  visit(ctx.statement).asInstanceOf[LogicalPlan]
}

// AbstractParseTreeVisitor
public T visit(ParseTree tree) {
  return tree.accept(this);
}

// XXXXContext
public <T> T accept(ParseTreeVisitor<? extends T> visitor) {
  if (visitor instanceof SqlBaseVisitor) return ((SqlBaseVisitor<? extends T>)visitor).visitXXXX(this);
  else return visitor.visitChildren(this);
}
```

----

**特质`ParserInterface`定义了将ANTLR4解析树结点转换为逻辑计划的抽象方法**，如下所示

- `parsePlan()`: 将字符串类型的SQL语句转化为逻辑计划

- `parseExpression()`: 将字符串类型的SQL语句转化为表达式
- `parseTableIdentifier()`: 将字符串类型的SQL语句转化为数据表标识
- `parseFunctionIdentifier()`: 将字符串类型的SQL语句转化为数据库定义函数的标识
- `parseTableSchema()`: 将字符串类型的SQL语句转化为表的schema信息，即结构体类型
- `parseDataType()`: : 将字符串类型的SQL语句转化为类型信息

**`AbstractSqlParser`继承了`ParserInterface`特质并实现了这些抽象方法**，下面以`parsePlan()`为例

- 调用了`parse()`方法构造`SqlBaseParser`对象去驱动访问者模式

- 调用了`AstBuilder.visitSingleStatement()`方法去解析逻辑计划

```scala
override def parsePlan(sqlText: String): LogicalPlan = parse(sqlText) { parser =>
  astBuilder.visitSingleStatement(parser.singleStatement()) match {
    case plan: LogicalPlan => plan
    case _ =>
    val position = Origin(None, None)
    throw new ParseException(Option(sqlText), "Unsupported SQL statement", position, position)
  }
}
```


### SparkSqlAstBuilder&SparkSqlParser

`SparkSqlAstBuilder`继承了`AstBuilder`，并在其基础上定义了一些 DDL 语句的访问操作，**实现了`SqlBaseBaseVistor`其中的剩余的访问者方法，由其实例在`SparkSqlParser`调用**

`SparkSqlParser`继承了`AbstractSqlParser`，并且重写了`parse()`方法，对字符串增加了对`${var}$`这些字符串的替换

```scala
class SparkSqlParser(conf: SQLConf) extends AbstractSqlParser(conf) {
  val astBuilder = new SparkSqlAstBuilder(conf)

  private val substitutor = new VariableSubstitution(conf)

  protected override def parse[T](command: String)(toResult: SqlBaseParser => T): T = {
    super.parse(substitutor.substitute(command))(toResult)
  }
}
```

### 示例

如图所示，下面是SQL语句`SELECT NAME FROM STUDENT WHERE AGE > 18 ORDER BY ID DESC`的ANTLR4解析树

![]({{ site.url }}/assets/img/2020-10-21-5.png)

由以下流程图所示，展示了此SQL语句的ANTLR4解析树结点转换为unresolved逻辑计划的调用逻辑，有以下符号标记

- `astBuilder`表示`SparkSqlAstBuilder`实例，继承自ANTLR产生的`SqlBaseVisitor`访问者类
- `sparkParser`表示`SparkSqlParser`实例，内部使用了`astBuilder`
- `parser`表示`SqlBaseParser`实例
- `tree`表示`ParseTree`实例，其子类表示各结点上下文如`StatementDefaultContext`

![]({{ site.url }}/assets/img/2020-10-21-6.png)

最终，ANTLR4解析树被转化为如下所示的Unsolved逻辑计划树

```
== Parsed Logical Plan ==
'Sort ['ID DESC NULLS LAST], true
+- 'Project ['NAME]
   +- 'Filter ('AGE > 18)
      +- 'UnresolvedRelation `STUDENT`
```

可以看出逻辑计划树和ANTLR4解析树是相对应的，`Sort`作为根结点，子结点的解析逻辑如下图所示，结合具体源码就可以看出解析逻辑

![]({{ site.url }}/assets/img/2020-10-21-7.png)

## REFERENCE
1. Spark SQL内核剖析
2. [Spark Sql LogicalPlan 介绍——zhmin](https://zhmin.github.io/2019/06/18/spark-sql-logicalplan/)
