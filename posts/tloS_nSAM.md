---
title: '第十章 Table API 与 SQL'
date: 2021-04-17 10:43:29
tags: [flink]
published: true
hideInList: false
feature: /post-images/tloS_nSAM.png
isTop: false
---
<p style="text-indent:2em">Table API 是流处理和批处理通用的关系型 API， Table API 可以基于流输入或者批输入来运行而不需要进行任何修改。 Table API 是 SQL 语言的超集并专门为 Apache Flink 设计的， Table API 是 Scala 和 Java 语言集成式的 API。 与常规 SQL 语言中将查询指定为字符串不同， Table API 查询是以 Java 或 Scala 中的语言嵌入样式来定义
的， 具有 IDE 支持如:自动完成和语法检测。</p>

## 需要引入的 pom 依赖

如果你想在 IDE 本地运行你的程序，你需要添加下面的模块，具体用哪个取决于你使用哪个 Planner：

```java
<!-- Either... (for the old planner that was available before Flink 1.9) -->
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-table-planner_2.12</artifactId>
    <version>1.10.1</version>
</dependency>
<!-- or.. (for the new Blink planner) -->
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-table-planner-blink_2.12</artifactId>
  <version>1.10.1</version>
  <scope>provided</scope>
</dependency>
```
此外，根据目标编程语言，您需要添加Java或Scala API。
```java
<!-- Either... -->
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-table-api-java-bridge_2.11</artifactId>
  <version>1.8.0</version>
</dependency>
<!-- or... -->
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-table-api-scala-bridge_2.11</artifactId>
  <version>1.8.0</version>
</dependency>
```
在内部，表生态系统的一部分是在Scala中实现的。 因此，请确保为批处理和流应用程序添加以下依赖项：
```java
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-streaming-scala_2.11</artifactId>
  <version>1.8.0</version>
</dependency>
```
如果要实现与Kafka或一组用户定义函数交互的自定义格式，以下依赖关系就足够了，可用于SQL客户端的JAR文件：
```java
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-table-common</artifactId>
  <version>1.8.0</version>
</dependency>
```
目前，该模块包括以下扩展点：
* SerializationSchemaFactory
* DeserializationSchemaFactory
* ScalarFunction
* TableFunction
* AggregateFunction

## 两种计划器（Planner）的主要区别

1. Blink 将批处理作业视作流处理的一种特例。严格来说，`Table` 和 `DataSet` 之间不支持相互转换，并且批处理作业也不会转换成 `DataSet` 程序而是转换成 `DataStream` 程序，流处理作业也一样。
2. Blink 计划器不支持 `BatchTableSource`，而是使用有界的 `StreamTableSource` 来替代。
3. 旧计划器和 Blink 计划器中 `FilterableTableSource` 的实现是不兼容的。旧计划器会将 `PlannerExpression` 下推至 `FilterableTableSource`，而 Blink 计划器则是将 `Expression` 下推。
4. 基于字符串的键值配置选项仅在 Blink 计划器中使用。（详情参见 [配置](https://ci.apache.org/projects/flink/flink-docs-release-1.12/zh/dev/table/config.html) ）
5. `PlannerConfig` 在两种计划器中的实现（`CalciteConfig`）是不同的。
6. Blink 计划器会将多sink（multiple-sinks）优化成一张有向无环图（DAG），`TableEnvironment` 和 `StreamTableEnvironment` 都支持该特性。旧计划器总是将每个sink都优化成一个新的有向无环图，且所有图相互独立。
7. 旧计划器目前不支持 catalog 统计数据，而 Blink 支持。

## 简单了解 TableAPI

所有用于批处理和流处理的 Table API 和 SQL 程序都遵循相同的模式。下面的代码示例展示了 Table API 和 SQL 程序的通用结构。

```java
public static void main(String[] args) throws Exception {
    // 对于批处理程序来说使用 ExecutionEnvironment 来替换 StreamExecutionEnvironment
    StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

    // 创建一个TableEnvironment
    // 对于批处理程序来说使用 BatchTableEnvironment 替换 StreamTableEnvironment
    StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

    // 注册一个 Table
    tableEnv.registerTable("table1", ...)            // 或者
    tableEnv.registerTableSource("table2", ...);     // 或者
    tableEnv.registerExternalCatalog("extCat", ...);

    // 从Table API的查询中创建一个Table
    Table tapiResult = tableEnv.scan("table1").select(...);
    // 从SQL查询中创建一个Table
    Table sqlResult  = tableEnv.sql("SELECT ... FROM table2 ... ");

    // 将Table API 种的结果 Table 发射到TableSink中 , SQL查询也是一样的
    tapiResult.writeToSink(...);

    // 执行
    env.execute();
}
```
## 创建 TableEnvironment

`TableEnvironment` 是 Table API 和 SQL 的核心概念。它负责:

- 在内部的 catalog 中注册 `Table`
- 注册外部的 catalog
- 加载可插拔模块
- 执行 SQL 查询
- 注册自定义函数 （scalar、table 或 aggregation）
- 将 `DataStream` 或 `DataSet` 转换成 `Table`
- 持有对 `ExecutionEnvironment` 或 `StreamExecutionEnvironment` 的引用

`Table` 总是与特定的 `TableEnvironment` 绑定。不能在同一条查询中使用不同 TableEnvironment 中的表，例如，对它们进行 join 或 union 操作。

`TableEnvironment` 可以通过静态方法 `BatchTableEnvironment.create()` 或者 `StreamTableEnvironment.create()` 在 `StreamExecutionEnvironment` 或者 `ExecutionEnvironment` 中创建，`TableConfig` 是可选项。`TableConfig`可用于配置`TableEnvironment`或定制的查询优化和转换过程(参见 [查询优化](https://ci.apache.org/projects/flink/flink-docs-release-1.12/zh/dev/table/common.html#query-optimization))。

请确保选择与你的编程语言匹配的特定的计划器`BatchTableEnvironment`/`StreamTableEnvironment`。

如果两种计划器的 jar 包都在 classpath 中（默认行为），你应该明确地设置要在当前程序中使用的计划器。

```java
// **********************
// FLINK STREAMING QUERY
// **********************
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.table.api.EnvironmentSettings;
import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;

EnvironmentSettings fsSettings = EnvironmentSettings.newInstance().useOldPlanner().inStreamingMode().build();
StreamExecutionEnvironment fsEnv = StreamExecutionEnvironment.getExecutionEnvironment();
StreamTableEnvironment fsTableEnv = StreamTableEnvironment.create(fsEnv, fsSettings);
// or TableEnvironment fsTableEnv = TableEnvironment.create(fsSettings);

// ******************
// FLINK BATCH QUERY
// ******************
import org.apache.flink.api.java.ExecutionEnvironment;
import org.apache.flink.table.api.bridge.java.BatchTableEnvironment;

ExecutionEnvironment fbEnv = ExecutionEnvironment.getExecutionEnvironment();
BatchTableEnvironment fbTableEnv = BatchTableEnvironment.create(fbEnv);

// **********************
// BLINK STREAMING QUERY
// **********************
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.table.api.EnvironmentSettings;
import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;

StreamExecutionEnvironment bsEnv = StreamExecutionEnvironment.getExecutionEnvironment();
EnvironmentSettings bsSettings = EnvironmentSettings.newInstance().useBlinkPlanner().inStreamingMode().build();
StreamTableEnvironment bsTableEnv = StreamTableEnvironment.create(bsEnv, bsSettings);
// or TableEnvironment bsTableEnv = TableEnvironment.create(bsSettings);

// ******************
// BLINK BATCH QUERY
// ******************
import org.apache.flink.table.api.EnvironmentSettings;
import org.apache.flink.table.api.TableEnvironment;

EnvironmentSettings bbSettings = EnvironmentSettings.newInstance().useBlinkPlanner().inBatchMode().build();
TableEnvironment bbTableEnv = TableEnvironment.create(bbSettings);
```

## 在 Catalog 中创建表

`TableEnvironment` 维护着一个由标识符（identifier）创建的表 catalog 的映射。标识符由三个部分组成：catalog 名称、数据库名称以及对象名称。如果 catalog 或者数据库没有指明，就会使用当前默认值（参见[表标识符扩展](https://ci.apache.org/projects/flink/flink-docs-release-1.12/zh/dev/table/common.html#expanding-table-identifiers)章节中的例子）。

`Table` 可以是虚拟的（视图 `VIEWS`）也可以是常规的（表 `TABLES`）。视图 `VIEWS`可以从已经存在的`Table`中创建，一般是 Table API 或者 SQL 的查询结果。 表`TABLES`描述的是外部数据，例如文件、数据库表或者消息队列。



### 临时表（Temporary Table）和永久表（Permanent Table）

表可以是临时的，并与单个 Flink 会话（session）的生命周期相关，也可以是永久的，并且在多个 Flink 会话和群集（cluster）中可见。

永久表需要 [catalog](https://ci.apache.org/projects/flink/flink-docs-release-1.12/zh/dev/table/catalogs.html)（例如 Hive Metastore）以维护表的元数据。一旦永久表被创建，它将对任何连接到 catalog 的 Flink 会话可见且持续存在，直至被明确删除。

另一方面，临时表通常保存于内存中并且仅在创建它们的 Flink 会话持续期间存在。这些表对于其它会话是不可见的。它们不与任何 catalog 或者数据库绑定但可以在一个命名空间（namespace）中创建。即使它们对应的数据库被删除，临时表也不会被删除。



#### 屏蔽（Shadowing）

可以使用与已存在的永久表相同的标识符去注册临时表。临时表会屏蔽永久表，并且只要临时表存在，永久表就无法访问。所有使用该标识符的查询都将作用于临时表。

这可能对实验（experimentation）有用。它允许先对一个临时表进行完全相同的查询，例如只有一个子集的数据，或者数据是不确定的。一旦验证了查询的正确性，就可以对实际的生产表进行查询。



### 创建表

#### 虚拟表

在 SQL 的术语中，Table API 的对象对应于`视图`（虚拟表）。它封装了一个逻辑查询计划。它可以通过以下方法在 catalog 中创建：

```java
// get a TableEnvironment
TableEnvironment tableEnv = ...; // see "Create a TableEnvironment" section

// table is the result of a simple projection query 
Table projTable = tableEnv.from("X").select(...);

// register the Table projTable as table "projectedTable"
tableEnv.createTemporaryView("projectedTable", projTable);
```

**注意：** 从传统数据库系统的角度来看，`Table` 对象与 `VIEW` 视图非常像。也就是，定义了 `Table` 的查询是没有被优化的， 而且会被内嵌到另一个引用了这个注册了的 `Table`的查询中。如果多个查询都引用了同一个注册了的`Table`，那么它会被内嵌每个查询中并被执行多次， 也就是说注册了的`Table`的结果**不会**被共享（注：Blink 计划器的`TableEnvironment`会优化成只执行一次）。



#### Connector Tables

另外一个方式去创建 `TABLE` 是通过 [connector](https://ci.apache.org/projects/flink/flink-docs-release-1.12/zh/dev/table/connect.html) 声明。Connector 描述了存储表数据的外部系统。存储系统例如 Apache Kafka 或者常规的文件系统都可以通过这种方式来声明。

```java
tableEnvironment
  .connect(...)
  .withFormat(...)
  .withSchema(...)
  .inAppendMode()
  .createTemporaryTable("MyTable")
```

### 扩展表标识符

表总是通过三元标识符注册，包括 catalog 名、数据库名和表名。

用户可以指定一个 catalog 和数据库作为 “当前catalog” 和”当前数据库”。有了这些，那么刚刚提到的三元标识符的前两个部分就可以被省略了。如果前两部分的标识符没有指定， 那么会使用当前的 catalog 和当前数据库。用户也可以通过 Table API 或 SQL 切换当前的 catalog 和当前的数据库。

标识符遵循 SQL 标准，因此使用时需要用反引号（```）进行转义。

```java
TableEnvironment tEnv = ...;
tEnv.useCatalog("custom_catalog");
tEnv.useDatabase("custom_database");

Table table = ...;

// register the view named 'exampleView' in the catalog named 'custom_catalog'
// in the database named 'custom_database' 
tableEnv.createTemporaryView("exampleView", table);

// register the view named 'exampleView' in the catalog named 'custom_catalog'
// in the database named 'other_database' 
tableEnv.createTemporaryView("other_database.exampleView", table);

// register the view named 'example.View' in the catalog named 'custom_catalog'
// in the database named 'custom_database' 
tableEnv.createTemporaryView("`example.View`", table);

// register the view named 'exampleView' in the catalog named 'other_catalog'
// in the database named 'other_database' 
tableEnv.createTemporaryView("other_catalog.other_database.exampleView", table);
```

## 查询表



### Table API

Table API 是关于 Scala 和 Java 的集成语言式查询 API。与 SQL 相反，Table API 的查询不是由字符串指定，而是在宿主语言中逐步构建。

Table API 是基于 `Table` 类的，该类表示一个表（流或批处理），并提供使用关系操作的方法。这些方法返回一个新的 Table 对象，该对象表示对输入 Table 进行关系操作的结果。 一些关系操作由多个方法调用组成，例如 `table.groupBy(...).select()`，其中 `groupBy(...)` 指定 `table` 的分组，而 `select(...)` 在 `table` 分组上的投影。

文档 [Table API](https://ci.apache.org/projects/flink/flink-docs-release-1.12/zh/dev/table/tableApi.html) 说明了所有流处理和批处理表支持的 Table API 算子。

以下示例展示了一个简单的 Table API 聚合查询：

```java
// get a TableEnvironment
TableEnvironment tableEnv = ...; // see "Create a TableEnvironment" section

// register Orders table

// scan registered Orders table
Table orders = tableEnv.from("Orders");
// compute revenue for all customers from France
Table revenue = orders
  .filter($("cCountry").isEqual("FRANCE"))
  .groupBy($("cID"), $("cName")
  .select($("cID"), $("cName"), $("revenue").sum().as("revSum"));

// emit or convert Table
// execute query
```

### SQL

Flink SQL 是基于实现了SQL标准的 [Apache Calcite](https://calcite.apache.org/) 的。SQL 查询由常规字符串指定。

文档 [SQL](https://ci.apache.org/projects/flink/flink-docs-release-1.12/zh/dev/table/sql/) 描述了Flink对流处理和批处理表的SQL支持。

下面的示例演示了如何指定查询并将结果作为 `Table` 对象返回。

```java
// get a TableEnvironment
TableEnvironment tableEnv = ...; // see "Create a TableEnvironment" section

// register Orders table

// compute revenue for all customers from France
Table revenue = tableEnv.sqlQuery(
    "SELECT cID, cName, SUM(revenue) AS revSum " +
    "FROM Orders " +
    "WHERE cCountry = 'FRANCE' " +
    "GROUP BY cID, cName"
  );

// emit or convert Table
// execute query
```

如下的示例展示了如何指定一个更新查询，将查询的结果插入到已注册的表中。

```java
// get a TableEnvironment
TableEnvironment tableEnv = ...; // see "Create a TableEnvironment" section

// register "Orders" table
// register "RevenueFrance" output table

// compute revenue for all customers from France and emit to "RevenueFrance"
tableEnv.executeSql(
    "INSERT INTO RevenueFrance " +
    "SELECT cID, cName, SUM(revenue) AS revSum " +
    "FROM Orders " +
    "WHERE cCountry = 'FRANCE' " +
    "GROUP BY cID, cName"
  );
```

### 混用 Table API 和 SQL

Table API 和 SQL 查询的混用非常简单因为它们都返回 `Table` 对象：

- 可以在 SQL 查询返回的 `Table` 对象上定义 Table API 查询。
- 在 `TableEnvironment` 中注册的[结果表](https://ci.apache.org/projects/flink/flink-docs-release-1.12/zh/dev/table/common.html#register-a-table)可以在 SQL 查询的 `FROM` 子句中引用，通过这种方法就可以在 Table API 查询的结果上定义 SQL 查询。

## 输出表

`Table` 通过写入 `TableSink` 输出。`TableSink` 是一个通用接口，用于支持多种文件格式（如 CSV、Apache Parquet、Apache Avro）、存储系统（如 JDBC、Apache HBase、Apache Cassandra、Elasticsearch）或消息队列系统（如 Apache Kafka、RabbitMQ）。

批处理 `Table` 只能写入 `BatchTableSink`，而流处理 `Table` 需要指定写入 `AppendStreamTableSink`，`RetractStreamTableSink` 或者 `UpsertStreamTableSink`。

请参考文档 [Table Sources & Sinks](https://ci.apache.org/projects/flink/flink-docs-release-1.12/zh/dev/table/sourceSinks.html) 以获取更多关于可用 Sink 的信息以及如何自定义 `TableSink`。

方法 `Table.executeInsert(String tableName)` 将 `Table` 发送至已注册的 `TableSink`。该方法通过名称在 catalog 中查找 `TableSink` 并确认`Table` schema 和 `TableSink` schema 一致。

下面的示例演示如何输出 `Table`：

```java
// get a TableEnvironment
TableEnvironment tableEnv = ...; // see "Create a TableEnvironment" section

// create an output Table
final Schema schema = new Schema()
    .field("a", DataTypes.INT())
    .field("b", DataTypes.STRING())
    .field("c", DataTypes.BIGINT());

tableEnv.connect(new FileSystem().path("/path/to/file"))
    .withFormat(new Csv().fieldDelimiter('|').deriveSchema())
    .withSchema(schema)
    .createTemporaryTable("CsvSinkTable");

// compute a result Table using Table API operators and/or SQL queries
Table result = ...
// emit the result Table to the registered TableSink
result.executeInsert("CsvSinkTable");
```

## 翻译与执行查询

两种计划器翻译和执行查询的方式是不同的。

- [**Blink planner**](https://ci.apache.org/projects/flink/flink-docs-release-1.12/zh/dev/table/common.html#tab_Blink_planner_9)
- [**Old planner**](https://ci.apache.org/projects/flink/flink-docs-release-1.12/zh/dev/table/common.html#tab_Old_planner_9)

不论输入数据源是流式的还是批式的，Table API 和 SQL 查询都会被转换成 [DataStream](https://ci.apache.org/projects/flink/flink-docs-release-1.12/zh/dev/datastream_api.html) 程序。查询在内部表示为逻辑查询计划，并被翻译成两个阶段：

1. 优化逻辑执行计划
2. 翻译成 DataStream 程序

Table API 或者 SQL 查询在下列情况下会被翻译：

- 当 `TableEnvironment.executeSql()` 被调用时。该方法是用来执行一个 SQL 语句，一旦该方法被调用， SQL 语句立即被翻译。
- 当 `Table.executeInsert()` 被调用时。该方法是用来将一个表的内容插入到目标表中，一旦该方法被调用， TABLE API 程序立即被翻译。
- 当 `Table.execute()` 被调用时。该方法是用来将一个表的内容收集到本地，一旦该方法被调用， TABLE API 程序立即被翻译。
- 当 `StatementSet.execute()` 被调用时。`Table` （通过 `StatementSet.addInsert()` 输出给某个 `Sink`）和 INSERT 语句 （通过调用 `StatementSet.addInsertSql()`）会先被缓存到 `StatementSet` 中，`StatementSet.execute()` 方法被调用时，所有的 sink 会被优化成一张有向无环图。
- 当 `Table` 被转换成 `DataStream` 时（参阅[与 DataStream 和 DataSet API 结合](https://ci.apache.org/projects/flink/flink-docs-release-1.12/zh/dev/table/common.html#integration-with-datastream-and-dataset-api)）。转换完成后，它就成为一个普通的 DataStream 程序，并会在调用 `StreamExecutionEnvironment.execute()` 时被执行。

**注意** **从 1.11 版本开始，`sqlUpdate` 方法 和 `insertInto` 方法被废弃，从这两个方法构建的 Table 程序必须通过 `StreamTableEnvironment.execute()` 方法执行，而不能通过 `StreamExecutionEnvironment.execute()` 方法来执行。**



## 与 DataStream 和 DataSet API 结合

在流处理方面两种计划器都可以与 `DataStream` API 结合。只有旧计划器可以与 `DataSet API` 结合。在批处理方面，Blink 计划器不能同两种计划器中的任何一个结合。

**注意：** 下文讨论的 `DataSet` API 只与旧计划起有关。

Table API 和 SQL 可以被很容易地集成并嵌入到 [DataStream](https://ci.apache.org/projects/flink/flink-docs-release-1.12/zh/dev/datastream_api.html) 和 [DataSet](https://ci.apache.org/projects/flink/flink-docs-release-1.12/zh/dev/batch/) 程序中。例如，可以查询外部表（例如从 RDBMS），进行一些预处理，例如过滤，投影，聚合或与元数据 join，然后使用 DataStream 或 DataSet API（以及在这些 API 之上构建的任何库，例如 CEP 或 Gelly）。相反，也可以将 Table API 或 SQL 查询应用于 DataStream 或 DataSet 程序的结果。

这种交互可以通过 `DataStream` 或 `DataSet` 与 `Table` 的相互转化实现。本节我们会介绍这些转化是如何实现的。



### Scala 隐式转换

Scala Table API 含有对 `DataSet`、`DataStream` 和 `Table` 类的隐式转换。 通过为 Scala DataStream API 导入 `org.apache.flink.table.api.bridge.scala._` 包以及 `org.apache.flink.api.scala._` 包，可以启用这些转换。



### 通过 DataSet 或 DataStream 创建`视图`

在 `TableEnvironment` 中可以将 `DataStream` 或 `DataSet` 注册成视图。结果视图的 schema 取决于注册的 `DataStream` 或 `DataSet` 的数据类型。请参阅文档 [数据类型到 table schema 的映射](https://ci.apache.org/projects/flink/flink-docs-release-1.12/zh/dev/table/common.html#mapping-of-data-types-to-table-schema)获取详细信息。

**注意：** 通过 `DataStream` 或 `DataSet` 创建的视图只能注册成临时视图。

```java
// get StreamTableEnvironment
// registration of a DataSet in a BatchTableEnvironment is equivalent
StreamTableEnvironment tableEnv = ...; // see "Create a TableEnvironment" section

DataStream<Tuple2<Long, String>> stream = ...

// register the DataStream as View "myTable" with fields "f0", "f1"
tableEnv.createTemporaryView("myTable", stream);

// register the DataStream as View "myTable2" with fields "myLong", "myString"
tableEnv.createTemporaryView("myTable2", stream, $("myLong"), $("myString"));
```




### 将 DataStream 或 DataSet 转换成表

与在 `TableEnvironment` 注册 `DataStream` 或 `DataSet` 不同，DataStream 和 DataSet 还可以直接转换成 `Table`。如果你想在 Table API 的查询中使用表，这将非常便捷。

```java
// get StreamTableEnvironment
// registration of a DataSet in a BatchTableEnvironment is equivalent
StreamTableEnvironment tableEnv = ...; // see "Create a TableEnvironment" section

DataStream<Tuple2<Long, String>> stream = ...

// Convert the DataStream into a Table with default fields "f0", "f1"
Table table1 = tableEnv.fromDataStream(stream);

// Convert the DataStream into a Table with fields "myLong", "myString"
Table table2 = tableEnv.fromDataStream(stream, $("myLong"), $("myString"));
```



### 将表转换成 DataStream 或 DataSet

`Table` 可以被转换成 `DataStream` 或 `DataSet`。通过这种方式，定制的 DataSet 或 DataStream 程序就可以在 Table API 或者 SQL 的查询结果上运行了。

将 `Table` 转换为 `DataStream` 或者 `DataSet` 时，你需要指定生成的 `DataStream` 或者 `DataSet` 的数据类型，即，`Table` 的每行数据要转换成的数据类型。通常最方便的选择是转换成 `Row` 。以下列表概述了不同选项的功能：

- **Row**: 字段按位置映射，字段数量任意，支持 `null` 值，无类型安全（type-safe）检查。
- **POJO**: 字段按名称映射（POJO 必须按`Table` 中字段名称命名），字段数量任意，支持 `null` 值，无类型安全检查。
- **Case Class**: 字段按位置映射，不支持 `null` 值，有类型安全检查。
- **Tuple**: 字段按位置映射，字段数量少于 22（Scala）或者 25（Java），不支持 `null` 值，无类型安全检查。
- **Atomic Type**: `Table` 必须有一个字段，不支持 `null` 值，有类型安全检查。



#### 将表转换成 DataStream

流式查询（streaming query）的结果表会动态更新，即，当新纪录到达查询的输入流时，查询结果会改变。因此，像这样将动态查询结果转换成 DataStream 需要对表的更新方式进行编码。

将 `Table` 转换为 `DataStream` 有两种模式：

1. **Append Mode**: 仅当动态 `Table` 仅通过`INSERT`更改进行修改时，才可以使用此模式，即，它仅是追加操作，并且之前输出的结果永远不会更新。
2. **Retract Mode**: 任何情形都可以使用此模式。它使用 boolean 值对 `INSERT` 和 `DELETE` 操作的数据进行标记。

```java
// get StreamTableEnvironment. 
StreamTableEnvironment tableEnv = ...; // see "Create a TableEnvironment" section

// Table with two fields (String name, Integer age)
Table table = ...

// convert the Table into an append DataStream of Row by specifying the class
DataStream<Row> dsRow = tableEnv.toAppendStream(table, Row.class);

// convert the Table into an append DataStream of Tuple2<String, Integer> 
//   via a TypeInformation
TupleTypeInfo<Tuple2<String, Integer>> tupleType = new TupleTypeInfo<>(
  Types.STRING(),
  Types.INT());
DataStream<Tuple2<String, Integer>> dsTuple = 
  tableEnv.toAppendStream(table, tupleType);

// convert the Table into a retract DataStream of Row.
//   A retract stream of type X is a DataStream<Tuple2<Boolean, X>>. 
//   The boolean field indicates the type of the change. 
//   True is INSERT, false is DELETE.
DataStream<Tuple2<Boolean, Row>> retractStream = 
  tableEnv.toRetractStream(table, Row.class);
```

**注意：** 文档[动态表](https://ci.apache.org/projects/flink/flink-docs-release-1.12/zh/dev/table/streaming/dynamic_tables.html)给出了有关动态表及其属性的详细讨论。

**注意** **一旦 Table 被转化为 DataStream，必须使用 StreamExecutionEnvironment 的 execute 方法执行该 DataStream 作业。**



#### 将表转换成 DataSet

将 `Table` 转换成 `DataSet` 的过程如下：

```java
// get BatchTableEnvironment
BatchTableEnvironment tableEnv = BatchTableEnvironment.create(env);

// Table with two fields (String name, Integer age)
Table table = ...

// convert the Table into a DataSet of Row by specifying a class
DataSet<Row> dsRow = tableEnv.toDataSet(table, Row.class);

// convert the Table into a DataSet of Tuple2<String, Integer> via a TypeInformation
TupleTypeInfo<Tuple2<String, Integer>> tupleType = new TupleTypeInfo<>(
  Types.STRING(),
  Types.INT());
DataSet<Tuple2<String, Integer>> dsTuple = 
  tableEnv.toDataSet(table, tupleType);
```

**注意** **一旦 Table 被转化为 DataSet，必须使用 ExecutionEnvironment 的 execute 方法执行该 DataSet 作业。**



### 数据类型到 Table Schema 的映射

Flink 的 DataStream 和 DataSet APIs 支持多样的数据类型。例如 Tuple（Scala 内置以及Flink Java tuple）、POJO 类型、Scala case class 类型以及 Flink 的 Row 类型等允许嵌套且有多个可在表的表达式中访问的字段的复合数据类型。其他类型被视为原子类型。下面，我们讨论 Table API 如何将这些数据类型类型转换为内部 row 表示形式，并提供将 `DataStream` 转换成 `Table` 的样例。

数据类型到 table schema 的映射有两种方式：**基于字段位置**或**基于字段名称**。

**基于位置映射**

基于位置的映射可在保持字段顺序的同时为字段提供更有意义的名称。这种映射方式可用于*具有特定的字段顺序*的复合数据类型以及原子类型。如 tuple、row 以及 case class 这些复合数据类型都有这样的字段顺序。然而，POJO 类型的字段则必须通过名称映射（参见下一章）。可以将字段投影出来，但不能使用`as`重命名。

定义基于位置的映射时，输入数据类型中一定不能存在指定的名称，否则 API 会假定应该基于字段名称进行映射。如果未指定任何字段名称，则使用默认的字段名称和复合数据类型的字段顺序，或者使用 `f0` 表示原子类型。

```java
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = ...; // see "Create a TableEnvironment" section;

DataStream<Tuple2<Long, Integer>> stream = ...

// convert DataStream into Table with default field names "f0" and "f1"
Table table = tableEnv.fromDataStream(stream);

// convert DataStream into Table with field "myLong" only
Table table = tableEnv.fromDataStream(stream, $("myLong"));

// convert DataStream into Table with field names "myLong" and "myInt"
Table table = tableEnv.fromDataStream(stream, $("myLong"), $("myInt"));
```

**基于名称的映射**

基于名称的映射适用于任何数据类型包括 POJO 类型。这是定义 table schema 映射最灵活的方式。映射中的所有字段均按名称引用，并且可以通过 `as` 重命名。字段可以被重新排序和映射。

若果没有指定任何字段名称，则使用默认的字段名称和复合数据类型的字段顺序，或者使用 `f0` 表示原子类型。

```java
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = ...; // see "Create a TableEnvironment" section

DataStream<Tuple2<Long, Integer>> stream = ...

// convert DataStream into Table with default field names "f0" and "f1"
Table table = tableEnv.fromDataStream(stream);

// convert DataStream into Table with field "f1" only
Table table = tableEnv.fromDataStream(stream, $("f1"));

// convert DataStream into Table with swapped fields
Table table = tableEnv.fromDataStream(stream, $("f1"), $("f0"));

// convert DataStream into Table with swapped fields and field names "myInt" and "myLong"
Table table = tableEnv.fromDataStream(stream, $("f1").as("myInt"), $("f0").as("myLong"));
```



#### 原子类型

Flink 将基础数据类型（`Integer`、`Double`、`String`）或者通用数据类型（不可再拆分的数据类型）视为原子类型。原子类型的 `DataStream` 或者 `DataSet` 会被转换成只有一条属性的 `Table`。属性的数据类型可以由原子类型推断出，还可以重新命名属性。

```java
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = ...; // see "Create a TableEnvironment" section

DataStream<Long> stream = ...

// convert DataStream into Table with default field name "f0"
Table table = tableEnv.fromDataStream(stream);

// convert DataStream into Table with field name "myLong"
Table table = tableEnv.fromDataStream(stream, $("myLong"));
```



#### Tuple类型（Scala 和 Java）和 Case Class类型（仅 Scala）

Flink 支持 Scala 的内置 tuple 类型并给 Java 提供自己的 tuple 类型。两种 tuple 的 DataStream 和 DataSet 都能被转换成表。可以通过提供所有字段名称来重命名字段（基于位置映射）。如果没有指明任何字段名称，则会使用默认的字段名称。如果引用了原始字段名称（对于 Flink tuple 为`f0`、`f1` … …，对于 Scala tuple 为`_1`、`_2` … …），则 API 会假定映射是基于名称的而不是基于位置的。基于名称的映射可以通过 `as` 对字段和投影进行重新排序。

```java
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = ...; // see "Create a TableEnvironment" section

DataStream<Tuple2<Long, String>> stream = ...

// convert DataStream into Table with default field names "f0", "f1"
Table table = tableEnv.fromDataStream(stream);

// convert DataStream into Table with renamed field names "myLong", "myString" (position-based)
Table table = tableEnv.fromDataStream(stream, $("myLong"), $("myString"));

// convert DataStream into Table with reordered fields "f1", "f0" (name-based)
Table table = tableEnv.fromDataStream(stream, $("f1"), $("f0"));

// convert DataStream into Table with projected field "f1" (name-based)
Table table = tableEnv.fromDataStream(stream, $("f1"));

// convert DataStream into Table with reordered and aliased fields "myString", "myLong" (name-based)
Table table = tableEnv.fromDataStream(stream, $("f1").as("myString"), $("f0").as("myLong"));
```



#### POJO 类型 （Java 和 Scala）

Flink 支持 POJO 类型作为复合类型。确定 POJO 类型的规则记录在[这里](https://ci.apache.org/projects/flink/flink-docs-release-1.12/zh/dev/types_serialization.html#pojos).

在不指定字段名称的情况下将 POJO 类型的 `DataStream` 或 `DataSet` 转换成 `Table` 时，将使用原始 POJO 类型字段的名称。名称映射需要原始名称，并且不能按位置进行。字段可以使用别名（带有 `as` 关键字）来重命名，重新排序和投影。

```java
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = ...; // see "Create a TableEnvironment" section

// Person is a POJO with fields "name" and "age"
DataStream<Person> stream = ...

// convert DataStream into Table with default field names "age", "name" (fields are ordered by name!)
Table table = tableEnv.fromDataStream(stream);

// convert DataStream into Table with renamed fields "myAge", "myName" (name-based)
Table table = tableEnv.fromDataStream(stream, $("age").as("myAge"), $("name").as("myName"));

// convert DataStream into Table with projected field "name" (name-based)
Table table = tableEnv.fromDataStream(stream, $("name"));

// convert DataStream into Table with projected and renamed field "myName" (name-based)
Table table = tableEnv.fromDataStream(stream, $("name").as("myName"));
```



#### Row类型

`Row` 类型支持任意数量的字段以及具有 `null` 值的字段。字段名称可以通过 `RowTypeInfo` 指定，也可以在将 `Row` 的 `DataStream` 或 `DataSet` 转换为 `Table` 时指定。Row 类型的字段映射支持基于名称和基于位置两种方式。字段可以通过提供所有字段的名称的方式重命名（基于位置映射）或者分别选择进行投影/排序/重命名（基于名称映射）。

```java
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = ...; // see "Create a TableEnvironment" section

// DataStream of Row with two fields "name" and "age" specified in `RowTypeInfo`
DataStream<Row> stream = ...

// convert DataStream into Table with default field names "name", "age"
Table table = tableEnv.fromDataStream(stream);

// convert DataStream into Table with renamed field names "myName", "myAge" (position-based)
Table table = tableEnv.fromDataStream(stream, $("myName"), $("myAge"));

// convert DataStream into Table with renamed fields "myName", "myAge" (name-based)
Table table = tableEnv.fromDataStream(stream, $("name").as("myName"), $("age").as("myAge"));

// convert DataStream into Table with projected field "name" (name-based)
Table table = tableEnv.fromDataStream(stream, $("name"));

// convert DataStream into Table with projected and renamed field "myName" (name-based)
Table table = tableEnv.fromDataStream(stream, $("name").as("myName"));
```



## 查询优化

- [**Blink planner**](https://ci.apache.org/projects/flink/flink-docs-release-1.12/zh/dev/table/common.html#tab_Blink_planner_20)
- [**Old planner**](https://ci.apache.org/projects/flink/flink-docs-release-1.12/zh/dev/table/common.html#tab_Old_planner_20)

Apache Flink 使用并扩展了 Apache Calcite 来执行复杂的查询优化。 这包括一系列基于规则和成本的优化，例如：

- 基于 Apache Calcite 的子查询解相关
- 投影剪裁
- 分区剪裁
- 过滤器下推
- 子计划消除重复数据以避免重复计算
- 特殊子查询重写，包括两部分：
  - 将 IN 和 EXISTS 转换为 left semi-joins
  - 将 NOT IN 和 NOT EXISTS 转换为 left anti-join
- 可选 join 重新排序
  - 通过 `table.optimizer.join-reorder-enabled` 启用

**注意：** 当前仅在子查询重写的结合条件下支持 IN / EXISTS / NOT IN / NOT EXISTS。

优化器不仅基于计划，而且还基于可从数据源获得的丰富统计信息以及每个算子（例如 io，cpu，网络和内存）的细粒度成本来做出明智的决策。

高级用户可以通过 `CalciteConfig` 对象提供自定义优化，可以通过调用 `TableEnvironment＃getConfig＃setPlannerConfig` 将其提供给 TableEnvironment。



## 解释表

Table API 提供了一种机制来解释计算 `Table` 的逻辑和优化查询计划。 这是通过 `Table.explain()` 方法或者 `StatementSet.explain()` 方法来完成的。`Table.explain()` 返回一个 Table 的计划。`StatementSet.explain()` 返回多 sink 计划的结果。它返回一个描述三种计划的字符串：

1. 关系查询的抽象语法树（the Abstract Syntax Tree），即未优化的逻辑查询计划，
2. 优化的逻辑查询计划，以及
3. 物理执行计划。

可以用 `TableEnvironment.explainSql()` 方法和 `TableEnvironment.executeSql()` 方法支持执行一个 `EXPLAIN` 语句获取逻辑和优化查询计划，请参阅 [EXPLAIN](https://ci.apache.org/projects/flink/flink-docs-release-1.12/zh/dev/table/sql/explain.html) 页面.

以下代码展示了一个示例以及对给定 `Table` 使用 `Table.explain()` 方法的相应输出：

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
StreamTableEnvironment tEnv = StreamTableEnvironment.create(env);

DataStream<Tuple2<Integer, String>> stream1 = env.fromElements(new Tuple2<>(1, "hello"));
DataStream<Tuple2<Integer, String>> stream2 = env.fromElements(new Tuple2<>(1, "hello"));

// explain Table API
Table table1 = tEnv.fromDataStream(stream1, $("count"), $("word"));
Table table2 = tEnv.fromDataStream(stream2, $("count"), $("word"));
Table table = table1
  .where($("word").like("F%"))
  .unionAll(table2);
System.out.println(table.explain());
```

上述例子的结果是：

```java
== Abstract Syntax Tree ==
LogicalUnion(all=[true])
  LogicalFilter(condition=[LIKE($1, _UTF-16LE'F%')])
    FlinkLogicalDataStreamScan(id=[1], fields=[count, word])
  FlinkLogicalDataStreamScan(id=[2], fields=[count, word])

== Optimized Logical Plan ==
DataStreamUnion(all=[true], union all=[count, word])
  DataStreamCalc(select=[count, word], where=[LIKE(word, _UTF-16LE'F%')])
    DataStreamScan(id=[1], fields=[count, word])
  DataStreamScan(id=[2], fields=[count, word])

== Physical Execution Plan ==
Stage 1 : Data Source
	content : collect elements with CollectionInputFormat

Stage 2 : Data Source
	content : collect elements with CollectionInputFormat

	Stage 3 : Operator
		content : from: (count, word)
		ship_strategy : REBALANCE

		Stage 4 : Operator
			content : where: (LIKE(word, _UTF-16LE'F%')), select: (count, word)
			ship_strategy : FORWARD

			Stage 5 : Operator
				content : from: (count, word)
				ship_strategy : REBALANCE
```

以下代码展示了一个示例以及使用 `StatementSet.explain()` 的多 sink 计划的相应输出：

```java
EnvironmentSettings settings = EnvironmentSettings.newInstance().useBlinkPlanner().inStreamingMode().build();
TableEnvironment tEnv = TableEnvironment.create(settings);

final Schema schema = new Schema()
    .field("count", DataTypes.INT())
    .field("word", DataTypes.STRING());

tEnv.connect(new FileSystem().path("/source/path1"))
    .withFormat(new Csv().deriveSchema())
    .withSchema(schema)
    .createTemporaryTable("MySource1");
tEnv.connect(new FileSystem().path("/source/path2"))
    .withFormat(new Csv().deriveSchema())
    .withSchema(schema)
    .createTemporaryTable("MySource2");
tEnv.connect(new FileSystem().path("/sink/path1"))
    .withFormat(new Csv().deriveSchema())
    .withSchema(schema)
    .createTemporaryTable("MySink1");
tEnv.connect(new FileSystem().path("/sink/path2"))
    .withFormat(new Csv().deriveSchema())
    .withSchema(schema)
    .createTemporaryTable("MySink2");

StatementSet stmtSet = tEnv.createStatementSet();

Table table1 = tEnv.from("MySource1").where($("word").like("F%"));
stmtSet.addInsert("MySink1", table1);

Table table2 = table1.unionAll(tEnv.from("MySource2"));
stmtSet.addInsert("MySink2", table2);

String explanation = stmtSet.explain();
System.out.println(explanation);
```

多 sink 计划的结果是：

```java
== Abstract Syntax Tree ==
LogicalLegacySink(name=[MySink1], fields=[count, word])
+- LogicalFilter(condition=[LIKE($1, _UTF-16LE'F%')])
   +- LogicalTableScan(table=[[default_catalog, default_database, MySource1, source: [CsvTableSource(read fields: count, word)]]])

LogicalLegacySink(name=[MySink2], fields=[count, word])
+- LogicalUnion(all=[true])
   :- LogicalFilter(condition=[LIKE($1, _UTF-16LE'F%')])
   :  +- LogicalTableScan(table=[[default_catalog, default_database, MySource1, source: [CsvTableSource(read fields: count, word)]]])
   +- LogicalTableScan(table=[[default_catalog, default_database, MySource2, source: [CsvTableSource(read fields: count, word)]]])

== Optimized Logical Plan ==
Calc(select=[count, word], where=[LIKE(word, _UTF-16LE'F%')], reuse_id=[1])
+- TableSourceScan(table=[[default_catalog, default_database, MySource1, source: [CsvTableSource(read fields: count, word)]]], fields=[count, word])

LegacySink(name=[MySink1], fields=[count, word])
+- Reused(reference_id=[1])

LegacySink(name=[MySink2], fields=[count, word])
+- Union(all=[true], union=[count, word])
   :- Reused(reference_id=[1])
   +- TableSourceScan(table=[[default_catalog, default_database, MySource2, source: [CsvTableSource(read fields: count, word)]]], fields=[count, word])

== Physical Execution Plan ==
Stage 1 : Data Source
	content : collect elements with CollectionInputFormat

	Stage 2 : Operator
		content : CsvTableSource(read fields: count, word)
		ship_strategy : REBALANCE

		Stage 3 : Operator
			content : SourceConversion(table:Buffer(default_catalog, default_database, MySource1, source: [CsvTableSource(read fields: count, word)]), fields:(count, word))
			ship_strategy : FORWARD

			Stage 4 : Operator
				content : Calc(where: (word LIKE _UTF-16LE'F%'), select: (count, word))
				ship_strategy : FORWARD

				Stage 5 : Operator
					content : SinkConversionToRow
					ship_strategy : FORWARD

					Stage 6 : Operator
						content : Map
						ship_strategy : FORWARD

Stage 8 : Data Source
	content : collect elements with CollectionInputFormat

	Stage 9 : Operator
		content : CsvTableSource(read fields: count, word)
		ship_strategy : REBALANCE

		Stage 10 : Operator
			content : SourceConversion(table:Buffer(default_catalog, default_database, MySource2, source: [CsvTableSource(read fields: count, word)]), fields:(count, word))
			ship_strategy : FORWARD

			Stage 12 : Operator
				content : SinkConversionToRow
				ship_strategy : FORWARD

				Stage 13 : Operator
					content : Map
					ship_strategy : FORWARD

					Stage 7 : Data Sink
						content : Sink: CsvTableSink(count, word)
						ship_strategy : FORWARD

						Stage 14 : Data Sink
							content : Sink: CsvTableSink(count, word)
							ship_strategy : FORWARD
```

### 动态表

如果流中的数据类型是 case class 可以直接根据 case class 的结构生成 table
```java
tableEnv.fromDataStream(dataStream)
```
或者根据字段顺序单独命名
```java
tableEnv.fromDataStream(dataStream,’id,’timestamp .......)
```
最后的动态表可以转换为流进行输出
```java
table.toAppendStream[(String,String)]
```
### 字段
用一个单引放到字段前面来标识字段名, 如 ‘name , ‘id ,’amount 等
## TableAPI 的窗口聚合操作
### 通过一个例子了解 TableAPI
```java
package myflink.sql;
 
import org.apache.flink.api.java.DataSet;
import org.apache.flink.api.java.ExecutionEnvironment;
import org.apache.flink.table.api.Table;
import org.apache.flink.table.api.java.BatchTableEnvironment;
 
 
public class WordCountSQL {
 
    public static void main(String[] args) throws Exception {
 
        ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
        BatchTableEnvironment tEnv = BatchTableEnvironment.getTableEnvironment(env);
 
        DataSet<WC> input = env.fromElements(
                WC.of("hello", 1),
                WC.of("hqs", 1),
                WC.of("world", 1),
                WC.of("hello", 1)
        );
        //注册数据集
        tEnv.registerDataSet("WordCount", input, "word, frequency");
 
        //执行SQL，并结果集做为一个新表
        Table table = tEnv.sqlQuery("SELECT word, SUM(frequency) as frequency FROM WordCount GROUP BY word");
 
        DataSet<WC> result = tEnv.toDataSet(table, WC.class);
 
        result.print();
 
    }
 
    public static class WC {
        public String word; //hello
        public long frequency;
 
        //创建构造方法，让flink进行实例化
        public WC() {}
 
        public static WC of(String word, long frequency) {
            WC wc = new WC();
            wc.word = word;
            wc.frequency = frequency;
            return wc;
        }
 
        @Override
        public String toString() {
            return "WC " + word + " " + frequency;
        }
    }
 
}
```
### 关于 group by
1. 如果了使用 groupby， table 转换为流的时候只能用 toRetractDstream
```java
DataStream<Tuple2<Boolean, Tuple2<Integer, Integer>>> joinStream = bsTableEnv.toRetractStream(queryTable, Types.TUPLE(Types.INT, Types.INT));
```
2. toRetractDstream 得到的第一个 boolean 型字段标识 true 就是最新的数据
(Insert)， false 表示过期老数据(Delete)

3. 如果使用的 api 包括时间窗口， 那么窗口的字段必须出现在 groupBy 中。
 ```java
Table resultTable= dataTable
    .window( Tumble over 10.seconds on 'ts as 'tw )
    .groupBy('id, 'tw)
    .select('id, 'id.count)
 ```
### 关于时间窗口
1. 用到时间窗口， 必须提前声明时间字段， 如果是 processTime 直接在创建动
态表时进行追加就可以。
```java
Table dataTable= tableEnv.fromDataStream(dataStream, 'id,'temperature, 'ps.proctime)
```
1. 如果是 EventTime 要在创建动态表时声明
```java
Table dataTable= tableEnv.fromDataStream(dataStream, 'id,'temperature, 'ts.rowtime)
```
1. 滚动窗口可以使用 Tumble over 10000.millis on 来表示
```java
Table resultTable = dataTable
    .window( Tumble over 10.seconds on 'ts as 'tw )
    .groupBy('id, 'tw)
    .select('id, 'id.count)
```
## SQL 如何编写
从一个txt文件中读取数据，txt文件中包含id, 人字, 书名，价格信息。然后将数据注册成一个表，然后将这个表的结果进行统计，按人名统计出来这个人买书所花费的钱，将结果sink到一个文件中。上代码。
```java
package myflink.sql;
 
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.api.common.typeinfo.TypeInformation;
import org.apache.flink.api.java.DataSet;
import org.apache.flink.api.java.ExecutionEnvironment;
import org.apache.flink.api.java.operators.DataSource;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.table.api.Table;
import org.apache.flink.table.api.Types;
import org.apache.flink.table.api.java.BatchTableEnvironment;
import org.apache.flink.table.sinks.CsvTableSink;
import org.apache.flink.table.sinks.TableSink;
 
public class SQLFromFile {
 
    public static void main(String[] args) throws Exception {
        ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
 
        BatchTableEnvironment tableEnv = BatchTableEnvironment.getTableEnvironment(env);
 
        env.setParallelism(1);
        //读取文件
        DataSource<String> input = env.readTextFile("test.txt");
        //将读取到的文件进行输出
        input.print();
        //转换为DataSet
        DataSet<Orders> inputDataSet = input.map(new MapFunction<String, Orders>() {
            @Override
            public Orders map(String s) throws Exception {
                String[] splits = s.split(" ");
                return Orders.of(Integer.valueOf(splits[0]), String.valueOf(splits[1]), String.valueOf(splits[2]), Double.valueOf(splits[3]));
            }
        });
        //转换为table
        Table order = tableEnv.fromDataSet(inputDataSet);
        //注册Orders表名
        tableEnv.registerTable("Orders", order);
        Table nameResult = tableEnv.scan("Orders").select("name");
        //输出一下表
        nameResult.printSchema();
 
        //执行一下查询
        Table sqlQueryResult = tableEnv.sqlQuery("select name, sum(price) as total from Orders group by name order by total desc");
        //查询结果转换为DataSet
        DataSet<Result> result = tableEnv.toDataSet(sqlQueryResult, Result.class);
        result.print();
 
        //以tuple的方式进行输出
        result.map(new MapFunction<Result, Tuple2<String, Double>>() {
            @Override
            public Tuple2<String, Double> map(Result result) throws Exception {
                String name = result.name;
                Double total = result.total;
                return Tuple2.of(name, total);
            }
        }).print();
 
        TableSink sink  = new CsvTableSink("SQLText.txt", " | ");
 
        //设置字段名
        String[] filedNames = {"name", "total"};
        //设置字段类型
        TypeInformation[] filedTypes = {Types.STRING(), Types.DOUBLE()};
 
        tableEnv.registerTableSink("SQLTEXT", filedNames, filedTypes, sink);
 
        sqlQueryResult.insertInto("SQLTEXT");
 
        env.execute();
 
    }
 
    public static class Orders {
        public Integer id;
        public String name;
        public String book;
        public Double price;
 
        public Orders() {
            super();
        }
 
        public static Orders of(Integer id, String name, String book, Double price) {
            Orders orders = new Orders();
            orders.id = id;
            orders.name = name;
            orders.book = book;
            orders.price = price;
            return orders;
        }
    }
 
    public static class Result {
        public String name;
        public Double total;
 
        public Result() {
            super();
        }
 
        public static Result of(String name, Double total) {
            Result result = new Result();
            result.name = name;
            result.total = total;
            return result;
        }
    }
 
}
```
## 操作
Table API支持下面的操作，请注意并不是所有的操作都同时支持批程序和流程序，不支持的会被响应的标记出来。
### Scan, Projection, and Filter

| Operators                              | Description                                                  |
| -------------------------------------- | ------------------------------------------------------------ |
| **From** `Batch` `Streaming`           | 与SQL查询中的FROM子句类似。 执行已注册表的扫描。<br/> `Table orders = tableEnv.from("Orders");` |
| **Select** `Batch` `Streaming`         | 与SQL SELECT语句类似。 执行选择操作。<br/> `Table orders= tableEnv.from("Orders")`<br/> `Table result = orders.select($("a"), $("c").as("d"))` <br/>可以使用星号（***）作为通配符，选择表格中的所有列。<br/> `Table orders = tableEnv.from("Orders")`<br/> `Table result = orders.select($("*"))` |
| **As** `Batch` `Streaming`             | 重命名字段。 `Table orders = tableEnv.from("Orders").as('x, 'y, 'z, 't)` |
| **Where / Filter** `Batch` `Streaming` | 与SQL WHERE子句类似。 过滤掉未通过过滤谓词的行。<br/> `Table orders = tableEnv.from("Orders")` <br/>`Table result = orders.filter($("a").mod(2).isEqual(0))`<br /> or<br />`Table orders= tableEnv.from("Orders")` <br/>`Table result = orders.where($("b").isEqual("red"))` |

### Column Operations

| Operators                                   | Description                                                  |
| ------------------------------------------- | :----------------------------------------------------------- |
| **AddColumns** `Batch` `Streaming`          | 执行字段添加操作。如果添加的字段已经存在，它将引发异常。<br/> `Table orders = tableEnv.from("Orders");` <br/>`Table result = orders.addColumns(concat($("c"), "sunny"))` |
| **AddOrReplaceColumns** `Batch` `Streaming` | 执行字段添加操作。如果添加列名称与现有列名称相同，则现有字段将被替换。此外，如果添加的字段具有重复的字段名称，则使用最后一个。<br/> `Table orders = tableEnv.from("Orders");` <br/>`Table result = orders.addOrReplaceColumns(concat($("c"), "sunny").as("desc"))` |
| **DropColumns** `Batch` `Streaming`         | 执行字段删除操作。字段表达式应该是字段引用表达式，并且只能删除现有字段。<br/> `Table orders = tableEnv.from("Orders");` <br/>`Table result = orders.dropColumns($("b"), $("c"))` |
| **RenameColumns** `Batch` `Streaming`       | 执行字段重命名操作。字段表达式应该是别名表达式，并且只有现有字段可以重命名。<br/> `Table orders = tableEnv.from("Orders");` <br/>`Table result = orders.renameColumns($("b").as("b2"), $("c").as("c2"))` |

### Aggregations

| Operators                                              | Description                                                  |
| ------------------------------------------------------ | :----------------------------------------------------------- |
| **GroupBy聚合** `Batch` `Streaming` `Result Updating`  | 类似于SQL GROUP BY子句。使用以下正在运行的聚合运算符将分组键上的行分组，以逐行聚合行。<br/> `Table orders: Table = tableEnv.scan("Orders")`<br/> `Table result = orders.groupBy($("a")).select($("a"), $("b").sum().as("d"))` <br/>注意：对于流式查询，计算查询结果所需的状态可能会无限增长，具体取决于聚合类型和不同分组键的数量。 请提供具有有效保留间隔的查询配置，以防止状态过大。 请参阅[查询配置](https://blog.csdn.net/lp284558195/article/details/104609739) |
| **GroupBy窗口聚合** `Batch` `Streaming`                | 在组窗口可能的一个或多个分组key上对表进行分组和聚集。<br/> `val orders: Table = tableEnv.scan("Orders")` <br/>`Table result = orders`<br />    ` .window(Tumble.over(lit(5).minutes()).on($("rowtime")).as("w")) // define window`<br />    ` .groupBy($("a"), $("w")) // group by key and window    `<br />  `// access window properties and aggregate `<br />   ` .select( `<br />       ` $("a"), `<br />       `$("w").start(), `<br />       ` $("w").end(),   `<br />       ` $("w").rowtime(), `<br />       `$("b").sum().as("d") `<br />  `  );` |
| **Over 窗口聚合** `Batch` `Streaming`                  | 类似于SQL OVER子句。基于前一行和后一行的窗口（范围），为每一行计算窗口聚合。<br/> `Table orders = tableEnv.from("Orders");`<br /> `Table result = orders     // define window `<br />   ` .window( `<br />       ` Over `<br />         ` .partitionBy($("a")) `<br />         ` .orderBy($("rowtime"))   `<br />         ` .preceding(UNBOUNDED_RANGE)   `<br />         ` .following(CURRENT_RANGE) `<br />         `  .as("w"))     // sliding aggregate`<br />    ` .select( `<br />         ` $("a"),`<br />         `$("b").avg().over($("w")),`<br />         `$("b").max().over($("w")), `<br />         `$("b").min().over($("w"))`<br />   `  );`<br />**Note:**必须在同一窗口（即相同的分区，排序和范围）上定义所有聚合。当前，仅支持PRECEDING（无边界和有界）到CURRENT ROW范围的窗口。目前尚不支持带有FOLLOWING的范围。必须在单个[时间属性](http://www.lllpan.top/article/55)上指定ORDER BY。 |
| **Distinct聚合** `Batch` `Streaming` `Result Updating` | 类似于SQL DISTINCT AGGREGATION子句，例如COUNT（DISTINCT a）。不同的聚合声明聚合函数（内置或用户定义的）仅应用于不同的输入值。Distinct聚合可以用于GroupBy聚合，GroupBy窗口聚合和Over窗口聚合。<br/>`Table orders = tableEnv.from("Orders");`<br /> `// Distinct aggregation on group by`<br /> `Table groupByDistinctResult = orders `<br />   ` .groupBy($("a")) `<br />   ` .select($("a"), $("b").sum().distinct().as("d")); `<br />`// Distinct aggregation on time window group by`<br /> `Table groupByWindowDistinctResult = orders `<br />   ` .window(Tumble `<br />          `  .over(lit(5).minutes()) `<br />          `  .on($("rowtime"))  `<br />          ` .as("w")`<br />   `  )`<br />    ` .groupBy($("a"), $("w"))`<br />    ` .select($("a"), $("b").sum().distinct().as("d")); `<br />`// Distinct aggregation on over window`<br /> `Table result = orders`<br />     `.window(Over`<br />         ` .partitionBy($("a"))`<br />         ` .orderBy($("rowtime"))`<br />         `.preceding(UNBOUNDED_RANGE)`<br />         `.as("w"))`<br />    ` .select( `<br />         `$("a"), $("b").avg().distinct().over($("w")),`<br />         `$("b").max().over($("w")),`<br />         ` $("b").min().over($("w"))`<br />    ` ); `<br />用户定义的聚合函数也可以与DISTINCT修饰符一起使用。要仅针对不同值计算聚合结果，只需向聚合函数添加distinct修饰符即可。 <br />`Table orders = tEnv.from("Orders"); `<br /> `// Use distinct aggregation for user-defined aggregate functions`<br />` tEnv.registerFunction("myUdagg", new MyUdagg());`<br /> `orders.groupBy("users")`<br />     `.select( `<br />        `$("users"), `<br />        ` call("myUdagg", $("points")).distinct().as("myDistinctResult")  `<br />  ` );`<br /> **Note:**对于流式查询，计算查询结果所需的状态可能会无限增长，具体取决于聚合类型和不同分组键的数量。 请提供具有有效保留间隔的查询配置，以防止状态过大。 请参阅[查询配置](https://blog.csdn.net/lp284558195/article/details/104609739) |
| **Distinct** `Batch` `Streaming` `Result Updating`     | 类似于SQL DISTINCT子句。返回具有不同值组合的记录。<br /> `Table orders= tableEnv.from("Orders")` <br />`Table result = orders.distinct()`<br /> **Note:**对于流查询，根据查询字段的数量，计算查询结果所需的状态可能会无限增长。请提供具有有效保留间隔的查询配置，以防止出现过多的状态。如果启用了状态清除功能，那么distinct必须发出消息，以防止下游运算符过早地退出状态，这会导致distinct包含结果更新。有关详细信息，请参阅[查询配置](https://blog.csdn.net/lp284558195/article/details/104609739) |

### Joins

| Operators                                                    | Description                                                  |
| ------------------------------------------------------------ | :----------------------------------------------------------- |
| **Inner Join** `Batch` `Streaming`                           | 类似于SQL JOIN子句。连接两个表。两个表必须具有不同的字段名称，并且至少一个相等的联接谓词必须通过联接运算符或使用where或filter运算符进行定义。<br />`Table left = tableEnv.fromDataSet(ds1, "a, b, c");`<br />`Table right = tableEnv.fromDataSet(ds2, "d, e, f");`<br />`Table result = left.join(right) `<br />    `.where($("a").isEqual($("d")))`<br />    ` .select($("a"), $("b"), $("e"));`<br />Note:** 对于流查询，根据不同输入行的数量，计算查询结果所需的状态可能会无限增长。请提供具有有效保留间隔的查询配置，以防止出现过多的状态。有关详细信息，请参阅[查询配置](https://blog.csdn.net/lp284558195/article/details/104609739) |
| **Outer Join** `Batch` `Streaming` `Result Updating`         | 类似于SQL LEFT / RIGHT / FULL OUTER JOIN子句。连接两个表。两个表必须具有不同的字段名称，并且必须至少定义一个相等联接谓词。 <br />`Table left = tableEnv.fromDataSet(ds1, "a, b, c");`<br />`Table right = tableEnv.fromDataSet(ds2, "d, e, f");`<br />`Table leftOuterResult = left.leftOuterJoin(right, $("a").isEqual($("d"))) `<br />                           ` .select($("a"), $("b"), $("e"));`<br />` Table rightOuterResult = left.rightOuterJoin(right, $("a").isEqual($("d")))   `<br />                         ` .select($("a"), $("b"), $("e"));`<br />` Table fullOuterResult = left.fullOuterJoin(right, $("a").isEqual($("d"))) `<br />                           ` .select($("a"), $("b"), $("e"));`<br />**Note:** 对于流查询，根据不同输入行的数量，计算查询结果所需的状态可能会无限增长。请提供具有有效保留间隔的查询配置，以防止出现过多的状态。有关详细信息，请参阅[查询配置](https://blog.csdn.net/lp284558195/article/details/104609739) |
| **Inner/Outer Interval Join**`Batch` `Streaming`             | **Note:** 时间窗口连接是可以以流方式处理的常规子集连接。 时间窗口连接需要至少一个等连接和一个限制双方时间的连接条件。可以通过两个适当的范围谓词（<，<=，> =，>）或比较两个输入表的相同类型的时间属性（即处理时间或事件时间）的单个相等谓词来定义这种条件。 例如，以下是有效的窗口连接条件：<br /> `'ltime === 'rtime`<br /> `'ltime >= 'rtime && 'ltime < 'rtime + 10.minutes`  <br />`Table left = tableEnv.fromDataSet(ds1, $("a"), $("b"), $("c"), $("ltime").rowtime());`<br />`Table right = tableEnv.fromDataSet(ds2, $("d"), $("e"), $("f"), $("rtime").rowtime())); `<br />`Table result = left.join(right)`<br />   `.where( `<br />    `and(`<br />         `$("a").isEqual($("d")), `<br />         ` $("ltime").isGreaterOrEqual($("rtime").minus(lit(5).minutes())),`<br />         `$("ltime").isLess($("rtime").plus(lit(10).minutes())) `<br />    `)) `<br /> ` .select($("a"), $("b"), $("e"), $("ltime"));` |
| **Inner Join with Table Function (UDTF)** `Batch` `Streaming` | 使用表函数的结果与表连接。左(外)表的每一行都与表函数的相应调用产生的所有行连接在一起。如果其表函数调用返回空结果，则删除左(外)表的一行。<br /> `// register User-Defined Table Function`<br />` TableFunction<String> split = new MySplitUDTF();`<br />` tableEnv.registerFunction("split", split); `<br /> `// join`<br />` Table orders = tableEnv.from("Orders");`<br />`Table result = orders`<br />     `.joinLateral(call("split", $("c")).as("s", "t", "v"))`<br />     ` .select($("a"), $("b"), $("s"), $("t"), $("v"));` |
| **Left Outer Join with Table Function (UDTF)** `Batch` `Streaming` | 使用表函数的结果连接表。 左(外)表的每一行与表函数的相应调用产生的所有行连接。 如果表函数调用返回空结果，则保留相应的外部行，并使用空值填充结果。 **Note:**目前，表函数的左外连接只能为空或为true。<br />`// register User-Defined Table Function`<br />`TableFunction<String> split = new MySplitUDTF(); `<br />`tableEnv.registerFunction("split", split); `<br />`// join`<br />` Table orders = tableEnv.from("Orders");`<br />`Table result = orders`<br />     `.leftOuterJoinLateral(call("split", $("c")).as("s", "t", "v")) `<br />     ` .select($("a"), $("b"), $("s"), $("t"), $("v"));` |
| **Join with Temporal Table** `Streaming`                     | [时态表](http://www.lllpan.top/article/54)是跟踪其随时间变化的表。 时态表功能提供对特定时间点时态表状态的访问。使用时态表函数联接表的语法与使用表函数进行内部联接的语法相同。 **Note:**当前仅支持使用临时表的内部联接。 <br />`Table ratesHistory = tableEnv.from("RatesHistory");`<br />` // register temporal table function with a time attribute and primary key`<br />`TemporalTableFunction rates = ratesHistory.createTemporalTableFunction( `<br />   ` "r_proctime", `<br />   ` "r_currency");`<br />`tableEnv.registerFunction("rates", rates);`<br />` // join with "Orders" based on the time attribute and key`<br />` Table orders = tableEnv.from("Orders");`<br />`Table result = orders`<br />    ` .joinLateral(call("rates", $("o_proctime")), $("o_currency").isEqual($("r_currency")))` |

### 集合算子

| Operators                        | Description                                                  |
| -------------------------------- | :----------------------------------------------------------- |
| **Union** `Batch`                | 类似于SQL UNION子句。合并两个已删除重复记录的表，两个表必须具有相同的字段类型。<br /> `Table left = ds1.toTable(tableEnv, 'a, 'b, 'c)`<br /> ` Table right = ds2.toTable(tableEnv, 'a, 'b, 'c)`<br /> ` Table result = left.union(right)` |
| **UnionAll** `Batch` `Streaming` | 类似于SQL UNION ALL子句。合并两个表，两个表必须具有相同的字段类型。<br /> `Table left = ds1.toTable(tableEnv, 'a, 'b, 'c)`<br /> ` Table right = ds2.toTable(tableEnv, 'a, 'b, 'c)`<br /> ` Table result = left.unionAll(right)` |
| **Intersect** `Batch`            | 类似于SQL INTERSECT子句。相交返回两个表中都存在的记录。如果一个记录在一个或两个表中多次出现，则仅返回一次，即结果表中没有重复的记录。两个表必须具有相同的字段类型。<br /> `Table left = ds1.toTable(tableEnv, 'a, 'b, 'c)`<br /> ` Table right = ds2.toTable(tableEnv, 'a, 'b, 'c)`<br />` Table result = left.intersect(right)` |
| **IntersectAll** `Batch`         | 类似于SQL INTERSECT ALL子句。IntersectAll返回两个表中都存在的记录。如果一个记录在两个表中都存在一次以上，则返回的次数与两个表中存在的次数相同，即，结果表可能具有重复的记录。两个表必须具有相同的字段类型。 <br /> `Table left = ds1.toTable(tableEnv, 'a, 'b, 'c)`<br /> ` Table right = ds2.toTable(tableEnv, 'a, 'b, 'c)`<br /> ` Table result = left.intersectAll(right)` |
| **Minus** `Batch`                | 类似于SQL EXCEPT子句。Minus返回左表中右表中不存在的记录。左表中的重复记录只返回一次，即删除重复项。 两个表必须具有相同的字段类型。 <br /> `Table left = ds1.toTable(tableEnv, 'a, 'b, 'c)`<br /> ` Table right = ds2.toTable(tableEnv, 'a, 'b, 'c)`<br /> ` Table result = left.minus(right)` |
| **MinusAll** `Batch`             | 类似于SQL EXCEPT ALL子句。 MinusAll返回右表中不存在的记录。 在左表中出现n次并在右表中出现m次的记录返回（n-m）次，即，删除右表中出现的重复数。 两个表必须具有相同的字段类型。 <br /> `Table left = ds1.toTable(tableEnv, 'a, 'b, 'c)`<br /> ` Table right = ds2.toTable(tableEnv, 'a, 'b, 'c)`<br /> ` Table result = left.minusAll(right)` |
| **In** `Batch` `Streaming`       | 与SQL IN子句类似。 如果表达式存在于给定的表子查询中，则返回true。 子查询表必须包含一列。 此列必须与表达式具有相同的数据类型。<br /> `Table left = ds1.toTable(tableEnv, "a, b, c"); `<br /> `Table right = ds2.toTable(tableEnv, "a");`<br /> ` Table result = left.select($("a"), $("b"), $("c")).where($("a").in(right)); `<br />**Note:** 对于流查询，该操作将被重写为join and group操作。根据不同的输入行的数量，计算查询结果所需的状态可能会无限增长。请提供具有有效保留间隔的查询配置，以防止出现过多的状态。有关详细信息，请参阅[查询配置](https://blog.csdn.net/lp284558195/article/details/104609739) |

### OrderBy, Offset & Fetch

| Operators                  | Description                                                  |
| -------------------------- | :----------------------------------------------------------- |
| **Order By** `Batch`       | 类似于SQL ORDER BY子句。返回在所有并行分区上全局排序的记录。<br />`Table in = tableEnv.fromDataSet(ds, "a, b, c");`<br /> `Table result = in.orderBy($("a").asc()");` |
| **Offset & Fetch** `Batch` | 类似于SQL OFFSET和FETCH子句。偏移和提取限制了从排序结果返回的记录数。偏移和提取在技术上是Order By运算符的一部分，因此必须在其之前。<br />`Table in = tableEnv.fromDataSet(ds, "a, b, c");`<br />  `// returns the first 5 records from the sorted result`<br />`Table result1 = in.orderBy($("a").asc()).fetch(5);`<br /> ` // skips the first 3 records and returns all following records from the sorted result `<br />`Table result2 = in.orderBy($("a").asc()).offset(3); `<br />` // skips the first 10 records and returns the next 5 records from the sorted result `<br />`Table result3 = in.orderBy($("a").asc()).offset(10).fetch(5);` |

### Insert

| Operators                           | Description                                                  |
| ----------------------------------- | :----------------------------------------------------------- |
| **Insert Into** `Batch` `Streaming` | 与SQL查询中的INSERT INTO子句相似。在已插入的输出表中执行插入。 输出表必须在TableEnvironment中注册。此外，已注册表的模式必须与查询的模式匹配。 <br />`Table orders = tableEnv.from("Orders");`<br />` orders.executeInsert("OutOrders");` |

### Group Windows

组窗口根据时间或行计数（row-count ）间隔将行组聚合为有限组，并按组聚合函数。 对于批处理表，窗口是按时间间隔对记录进行分组的便捷快捷方式。

Windows是使用window（w：GroupWindow）子句定义的，并且需要使用as子句指定的别名。为了按窗口对表进行分组，必须像常规分组属性一样在groupBy（…）子句中引用窗口别名。

以下示例显示如何在表上定义窗口聚合。

```java
Table table = input
  .window([GroupWindow w].as("w"))  // define window with alias w
  .groupBy($("w"))  // group the table by window w
  .select($("b").sum());  // aggregate
```

在流式环境中，如果窗口聚合除了窗口之外还在一个或多个属性上进行分组，则它们只能并行计算。即groupBy（…）子句引用了窗口别名和至少一个其他属性。仅引用窗口别名的groupBy（…）子句（例如上例中的子句）只能由单个非并行任务求值。以下示例显示如何使用其他分组属性定义窗口聚合。

```java
Table table = input
  .window([GroupWindow w].as("w"))  // define window with alias w
  .groupBy($("w"), $("a"))  // group the table by attribute a and window w
  .select($("a"), $("b").sum());  // aggregate
```

可以在select语句中将窗口属性（例如时间窗口的开始，结束或行时间戳）添加为窗口别名的属性，分别为w.start，w.end和w.rowtime。窗口开始和行时间时间戳是包含窗口的上下边界。相反，窗口结束时间戳是唯一的窗口上边界。例如，从下午2点开始的30分钟滚动窗口将以14：00：00.000作为开始时间戳，以14：29：59.999作为行时间时间戳，以14：30：00.000作为结束时间戳。

```java
Table table = input
  .window([GroupWindow w].as("w"))  // define window with alias w
  .groupBy($("w"), $("a"))  // group the table by attribute a and window w
  .select($("a"), $("w").start(), $("w").end(), $("w").rowtime(), $("b").count()); // aggregate and add window start, end, and rowtime timestamps
```

Window参数定义如何将行映射到窗口。窗口不是用户可以实现的接口。相反，Table API提供了一组具有特定语义的预定义Window类，这些类被转换为基础的DataStream或DataSet操作。支持的窗口定义在下面列出。

#### Tumble (滚动窗口)

滚动窗口将行分配给固定长度的不重叠的连续窗口。

例如，5分钟的翻滚窗口以5分钟为间隔对行进行分组。 可以在事件时间，处理时间或行数上定义翻滚窗口。使用Tumble类定义翻滚窗口如下：

滚动窗口是使用Tumble类定义的，如下所示：

| Method | Description                                                  |
| ------ | :----------------------------------------------------------- |
| over   | 定义窗口的长度，可以是时间间隔也可以是行数间隔。             |
| on     | 时间属性为组（时间间隔）或排序（行计数）。 对于批处理查询，这可能是任何Long或Timestamp属性。 对于流式查询，这必须是声明的事件时间或处理时间属性。 |
| as     | 为窗口指定别名。 别名用于引用以下groupBy（）子句中的窗口，并可选择在select（）子句中选择窗口属性，如window start，end或rowtime timestamp。 |

```java
// Tumbling Event-time Window
.window(Tumble.over(lit(10).minutes()).on($("rowtime")).as("w"));

// Tumbling Processing-time Window (assuming a processing-time attribute "proctime")
.window(Tumble.over(lit(10).minutes()).on($("proctime")).as("w"));

// Tumbling Row-count Window (assuming a processing-time attribute "proctime")
.window(Tumble.over(rowInterval(10)).on($("proctime")).as("w"));
```

#### Slide (滑动窗口)

滑动窗口的大小固定，并以指定的滑动间隔滑动。如果滑动间隔小于窗口大小，则滑动窗口重叠。因此，可以将行分配给多个窗口。

例如，15分钟大小和5分钟滑动间隔的滑动窗口将每行分配给3个不同的15分钟大小的窗口，这些窗口以5分钟的间隔进行调用。 可以在事件时间，处理时间或行数上定义滑动窗口。

滑动窗口是通过使用Slide类定义的，如下所示：

| Method | Description                                                  |
| ------ | ------------------------------------------------------------ |
| over   | 定义窗口的长度，可以是时间或行计数间隔。                     |
| every  | 定义滑动间隔，可以是时间间隔也可以是行数。 滑动间隔必须与大小间隔的类型相同。 |
| on     | 时间属性为组（时间间隔）或排序（行计数）。 对于批处理查询，这可能是任何Long或Timestamp属性。 对于流式查询，这必须是[声明的事件时间或处理时间属性](https://ci.apache.org/projects/flink/flink-docs-release-1.10/dev/table/streaming/time_attributes.html)。 |
| as     | 为窗口指定别名。 别名用于引用以下groupBy（）子句中的窗口，并可选择在select（）子句中选择窗口属性，如window start，end或rowtime timestamp。 |

```java
// Sliding Event-time Window
.window(Slide.over(lit(10).minutes())
            .every(lit(5).minutes())
            .on($("rowtime"))
            .as("w"));

// Sliding Processing-time window (assuming a processing-time attribute "proctime")
.window(Slide.over(lit(10).minutes())
            .every(lit(5).minutes())
            .on($("proctime"))
            .as("w"));

// Sliding Row-count window (assuming a processing-time attribute "proctime")
.window(Slide.over(rowInterval(10)).every(rowInterval(5)).on($("proctime")).as("w"));
```

#### Session (会话窗口)

会话窗口没有固定的大小，但是其边界由不活动的时间间隔定义，即如果在定义的间隔时间内没有事件出现，则会话窗口关闭。

例如，间隔30分钟的会话窗口会在30分钟不活动后观察到一行（否则该行将被添加到现有窗口）后开始，如果30分钟内未添加任何行，则关闭该窗口。会话窗口可以在事件时间或处理时间工作。

会话窗口是通过使用Session类定义的，如下所示：

| Method  | Description                                                  |
| ------- | ------------------------------------------------------------ |
| withGap | 将两个窗口之间的间隔定义为时间间隔。                         |
| on      | 时间属性为组（时间间隔）或排序（行计数）。 对于批处理查询，这可能是任何Long或Timestamp属性。 对于流式查询，这必须是声明的事件时间或处理时间属性。 |
| as      | 为窗口指定别名。 别名用于引用以下groupBy（）子句中的窗口，并可选择在select（）子句中选择窗口属性，如window start，end或rowtime timestamp。 |

```java
// Session Event-time Window
.window(Session.withGap(lit(10).minutes()).on($("rowtime")).as("w"));

// Session Processing-time Window (assuming a processing-time attribute "proctime")
.window(Session.withGap(lit(10).minutes()).on($("proctime")).as("w"));
```

### Over Windows

窗口聚合是标准SQL（OVER子句）已知的，并在查询的SELECT子句中定义。与在GROUP BY子句中指定的组窗口不同，在窗口上方不会折叠行。取而代之的是在窗口聚合中，为每个输入行在其相邻行的范围内计算聚合。

使用window（w：OverWindow *）子句（在Python API中使用over_window（* OverWindow））定义窗口，并在select() 方法中通过别名引用。以下示例显示如何在表上定义窗口聚合。

```java
Table table = input
  .window([OverWindow w].as("w"))           // define over window with alias w
  .select($("a"), $("b").sum().over($("w")), $("c").min().over($("w"))); // aggregate over the over window w
```

OverWindow定义了计算聚合的行范围。OverWindow不是用户可以实现的接口。相反，Table API提供了Over类来配置over窗口的属性。可以在事件时间或处理时间以及指定为时间间隔或行计数的范围上定义窗口上方。受支持的over窗口定义作为Over（和其他类）上的方法公开，并在下面列出：

| Method      | Required | Description                                                  |
| ----------- | -------- | ------------------------------------------------------------ |
| partitionBy | Optional | 在一个或多个属性上定义输入的分区。每个分区都经过单独排序，并且聚合函数分别应用于每个分区。 **Note:** 在流环境中，如果窗口包含partition by子句，则只能并行计算整个窗口聚合。没有partitionBy（…），流将由单个非并行任务处理。 |
| orderBy     | Required | 定义每个分区内的行顺序，从而定义将聚合函数应用于行的顺序。 **Note:** 对于流查询，它必须是声明的事件时间或处理时间时间属性。当前，仅支持单个sort属性。 |
| preceding   | Optional | 定义窗口中包含的并在当前行之前的行的间隔。该间隔可以指定为时间间隔或行计数间隔。 用时间间隔的大小指定窗口上的边界，例如，时间间隔为10分钟，行计数间隔为10行。 使用常数指定窗口上的无边界，即对于时间间隔为UNBOUNDED_RANGE或对于行计数间隔为UNBOUNDED_ROW。Windows上的无边界从分区的第一行开始。 如果省略了前面的子句，则将UNBOUNDED_RANGE和CURRENT_RANGE用作窗口的默认前后。 |
| following   | Optional | 定义窗口中包含并紧随当前行的行的窗口间隔。该间隔必须与前面的间隔（时间或行计数）以相同的单位指定。目前，不支持具有当前行之后的行的窗口。而是可以指定两个常量之一： 1. CURRENT_ROW将窗口的上限设置为当前行。 2. CURRENT_RANGE将窗口的上限设置为当前行的排序键，即，与当前行具有相同排序键的所有行都包含在窗口中。 如果省略以下子句，则将时间间隔窗口的上限定义为CURRENT_RANGE，将行计数间隔窗口的上限定义为CURRENT_ROW。 |
| as          | Required | 为上方窗口分配别名。别名用于引用以下select（）子句中的over窗口。 |

注意：当前，同一select（）调用中的所有聚合函数必须在相同的窗口范围内计算。

#### Unbounded Over Windows

```java
// Unbounded Event-time over window (assuming an event-time attribute "rowtime")
.window(Over.partitionBy($("a")).orderBy($("rowtime")).preceding(UNBOUNDED_RANGE).as("w"));

// Unbounded Processing-time over window (assuming a processing-time attribute "proctime")
.window(Over.partitionBy($("a")).orderBy("proctime").preceding(UNBOUNDED_RANGE).as("w"));

// Unbounded Event-time Row-count over window (assuming an event-time attribute "rowtime")
.window(Over.partitionBy($("a")).orderBy($("rowtime")).preceding(UNBOUNDED_ROW).as("w"));
 
// Unbounded Processing-time Row-count over window (assuming a processing-time attribute "proctime")
.window(Over.partitionBy($("a")).orderBy($("proctime")).preceding(UNBOUNDED_ROW).as("w"));
```

#### Bounded Over Windows

```java
// Bounded Event-time over window (assuming an event-time attribute "rowtime")
.window(Over.partitionBy($("a")).orderBy($("rowtime")).preceding(lit(1).minutes()).as("w"))

// Bounded Processing-time over window (assuming a processing-time attribute "proctime")
.window(Over.partitionBy($("a")).orderBy($("proctime")).preceding(lit(1).minutes()).as("w"))

// Bounded Event-time Row-count over window (assuming an event-time attribute "rowtime")
.window(Over.partitionBy($("a")).orderBy($("rowtime")).preceding(rowInterval(10)).as("w"))
 
// Bounded Processing-time Row-count over window (assuming a processing-time attribute "proctime")
.window(Over.partitionBy($("a")).orderBy($("proctime")).preceding(rowInterval(10)).as("w"))
```

### 基于行的操作

基于行的操作生成具有多列的输出。

- **Map** `Batch` `Streaming`

  使用用户定义的标量函数或内置标量函数执行映射操作。如果输出类型是复合类型，则输出将被展平。

  ```java
  public class MyMapFunction extends ScalarFunction {
      public Row eval(String a) {
          return Row.of(a, "pre-" + a);
      }
  
      @Override
      public TypeInformation<?> getResultType(Class<?>[] signature) {
          return Types.ROW(Types.STRING(), Types.STRING());
      }
  }
  
  ScalarFunction func = new MyMapFunction();
  tableEnv.registerFunction("func", func);
  
  Table table = input
    .map(call("func", $("c")).as("a", "b"))
  ```

- **FlatMap** `Batch` `Streaming`

  使用表函数执行flatMap操作。

  ```java
  public class MyFlatMapFunction extends TableFunction<Row> {
  
      public void eval(String str) {
          if (str.contains("#")) {
              String[] array = str.split("#");
              for (int i = 0; i < array.length; ++i) {
                  collect(Row.of(array[i], array[i].length()));
              }
          }
      }
  
      @Override
      public TypeInformation<Row> getResultType() {
          return Types.ROW(Types.STRING(), Types.INT());
      }
  }
  
  TableFunction func = new MyFlatMapFunction();
  tableEnv.registerFunction("func", func);
  
  Table table = input
    .flatMap(call("func", $("c")).as("a", "b"))
  ```

- **Aggregate** `Batch` `Streaming` `Result Updating`

  使用聚合函数执行聚合操作。您必须使用select语句关闭“聚合”，并且select语句不支持聚合函数。如果输出类型是复合类型，则聚合的输出将被展平。

  ```java
  public class MyMinMaxAcc {
      public int min = 0;
      public int max = 0;
  }
  
  public class MyMinMax extends AggregateFunction<Row, MyMinMaxAcc> {
  
      public void accumulate(MyMinMaxAcc acc, int value) {
          if (value < acc.min) {
              acc.min = value;
          }
          if (value > acc.max) {
              acc.max = value;
          }
      }
  
      @Override
      public MyMinMaxAcc createAccumulator() {
          return new MyMinMaxAcc();
      }
  
      public void resetAccumulator(MyMinMaxAcc acc) {
          acc.min = 0;
          acc.max = 0;
      }
  
      @Override
      public Row getValue(MyMinMaxAcc acc) {
          return Row.of(acc.min, acc.max);
      }
  
      @Override
      public TypeInformation<Row> getResultType() {
          return new RowTypeInfo(Types.INT, Types.INT);
      }
  }
  
  AggregateFunction myAggFunc = new MyMinMax();
  tableEnv.registerFunction("myAggFunc", myAggFunc);
  Table table = input
    .groupBy($("key"))
    .aggregate(call("myAggFunc", $("a")).as("x", "y"))
    .select($("key"), $("x"), $("y"))
  ```

- **Group Window Aggregate** `Batch` `Streaming`

  在组窗口和可能的一个或多个分组键上对表进行分组和聚集。您必须使用select语句关闭“聚合”。并且select语句不支持“ *”或聚合函数。

  ```java
  AggregateFunction myAggFunc = new MyMinMax();
  tableEnv.registerFunction("myAggFunc", myAggFunc);
  
  Table table = input
      .window(Tumble.over(lit(5).minutes())
                    .on($("rowtime"))
                    .as("w")) // define window
      .groupBy($("key"), $("w")) // group by key and window
      .aggregate(call("myAggFunc", $("a")).as("x", "y"))
      .select($("key"), $("x"), $("y"), $("w").start(), $("w").end()); // access window properties and aggregate results
  ```

- **FlatAggregate** `Batch` `Streaming`

  类似于GroupBy聚合。使用以下运行表聚合运算符将分组键上的行分组，以逐行聚合行。与AggregateFunction的区别在于TableAggregateFunction可以为一个组返回0个或更多记录。您必须使用select语句关闭“ flatAggregate”。并且select语句不支持聚合函数。除了使用emitValue来输出结果之外，还可以使用emitUpdateWithRetract方法。与emitValue不同，emitUpdateWithRetract用于发出已更新的值。此方法在撤消模式下增量输出数据，即，一旦有更新，我们就必须撤消旧记录，然后再发送新的更新记录。如果在表聚合函数中定义了这两种方法，则将优先使用emitUpdateWithRetract方法，因为这两种方法比emitValue更有效，因为它可以增量输出值。

  ```java
  /**
   * Accumulator for Top2.
   */
  public class Top2Accum {
      public Integer first;
      public Integer second;
  }
  
  /**
   * The top2 user-defined table aggregate function.
   */
  public class Top2 extends TableAggregateFunction<Tuple2<Integer, Integer>, Top2Accum> {
  
      @Override
      public Top2Accum createAccumulator() {
          Top2Accum acc = new Top2Accum();
          acc.first = Integer.MIN_VALUE;
          acc.second = Integer.MIN_VALUE;
          return acc;
      }
  
  
      public void accumulate(Top2Accum acc, Integer v) {
          if (v > acc.first) {
              acc.second = acc.first;
              acc.first = v;
          } else if (v > acc.second) {
              acc.second = v;
          }
      }
  
      public void merge(Top2Accum acc, java.lang.Iterable<Top2Accum> iterable) {
          for (Top2Accum otherAcc : iterable) {
              accumulate(acc, otherAcc.first);
              accumulate(acc, otherAcc.second);
          }
      }
  
      public void emitValue(Top2Accum acc, Collector<Tuple2<Integer, Integer>> out) {
          // emit the value and rank
          if (acc.first != Integer.MIN_VALUE) {
              out.collect(Tuple2.of(acc.first, 1));
          }
          if (acc.second != Integer.MIN_VALUE) {
              out.collect(Tuple2.of(acc.second, 2));
          }
      }
  }
  
  tEnv.registerFunction("top2", new Top2());
  Table orders = tableEnv.from("Orders");
  Table result = orders
      .groupBy($("key"))
      .flatAggregate(call("top2", $("a")).as("v", "rank"))
      .select($("key"), $("v"), $("rank");
  ```
  
  **Note:**对于流查询，根据聚合的类型和不同的分组键的数量，计算查询结果所需的状态可能会无限增长。请提供具有有效保留间隔的查询配置，以防止出现过多的状态。有关详细信息，请参见查询配置。
  
- **Group Window FlatAggregate** `Batch` `Streaming`

  在组窗口和可能的一个或多个分组键上对表进行分组和聚集。您必须使用select语句关闭“ flatAggregate”。并且select语句不支持聚合函数。

  ```java
  tableEnv.registerFunction("top2", new Top2());
  Table orders = tableEnv.from("Orders");
  Table result = orders
      .window(Tumble.over(lit(5).minutes())
                    .on($("rowtime"))
                    .as("w")) // define window
      .groupBy($("a"), $("w")) // group by key and window
      .flatAggregate(call("top2", $("b").as("v", "rank"))
      .select($("a"), $("w").start(), $("w").end(), $("w").rowtime(), $("v"), $("rank")); // access window properties and aggregate results
  ```

## Data Types

请参阅有关[数据类型](http://www.lllpan.top/article/52)的专用页面。通用类型和（嵌套的）复合类型（例如POJO，元组，行，Scala案例类）也可以是一行的字段。可以使用值访问功能访问具有任意嵌套的复合类型的字段。泛型类型被视为黑盒，可以通过用户定义的函数传递或处理。

## 表达式语法

前面几节中的某些运算符期望一个或多个表达式。可以使用嵌入式Scala DSL或字符串指定表达式。请参考上面的示例以了解如何指定表达式。

这是用于表达式的EBNF语法：

```sh
expressionList = expression , { "," , expression } ;

expression = overConstant | alias ;

alias = logic | ( logic , "as" , fieldReference ) | ( logic , "as" , "(" , fieldReference , { "," , fieldReference } , ")" ) ;

logic = comparison , [ ( "&&" | "||" ) , comparison ] ;

comparison = term , [ ( "=" | "==" | "===" | "!=" | "!==" | ">" | ">=" | "<" | "<=" ) , term ] ;

term = product , [ ( "+" | "-" ) , product ] ;

product = unary , [ ( "*" | "/" | "%") , unary ] ;

unary = [ "!" | "-" | "+" ] , composite ;

composite = over | suffixed | nullLiteral | prefixed | atom ;

suffixed = interval | suffixAs | suffixCast | suffixIf | suffixDistinct | suffixFunctionCall | timeIndicator ;

prefixed = prefixAs | prefixCast | prefixIf | prefixDistinct | prefixFunctionCall ;

interval = timeInterval | rowInterval ;

timeInterval = composite , "." , ("year" | "years" | "quarter" | "quarters" | "month" | "months" | "week" | "weeks" | "day" | "days" | "hour" | "hours" | "minute" | "minutes" | "second" | "seconds" | "milli" | "millis") ;

rowInterval = composite , "." , "rows" ;

suffixCast = composite , ".cast(" , dataType , ")" ;

prefixCast = "cast(" , expression , dataType , ")" ;

dataType = "BYTE" | "SHORT" | "INT" | "LONG" | "FLOAT" | "DOUBLE" | "BOOLEAN" | "STRING" | "DECIMAL" | "SQL_DATE" | "SQL_TIME" | "SQL_TIMESTAMP" | "INTERVAL_MONTHS" | "INTERVAL_MILLIS" | ( "MAP" , "(" , dataType , "," , dataType , ")" ) | ( "PRIMITIVE_ARRAY" , "(" , dataType , ")" ) | ( "OBJECT_ARRAY" , "(" , dataType , ")" ) ;

suffixAs = composite , ".as(" , fieldReference , ")" ;

prefixAs = "as(" , expression, fieldReference , ")" ;

suffixIf = composite , ".?(" , expression , "," , expression , ")" ;

prefixIf = "?(" , expression , "," , expression , "," , expression , ")" ;

suffixDistinct = composite , "distinct.()" ;

prefixDistinct = functionIdentifier , ".distinct" , [ "(" , [ expression , { "," , expression } ] , ")" ] ;

suffixFunctionCall = composite , "." , functionIdentifier , [ "(" , [ expression , { "," , expression } ] , ")" ] ;

prefixFunctionCall = functionIdentifier , [ "(" , [ expression , { "," , expression } ] , ")" ] ;

atom = ( "(" , expression , ")" ) | literal | fieldReference ;

fieldReference = "*" | identifier ;

nullLiteral = "nullOf(" , dataType , ")" ;

timeIntervalUnit = "YEAR" | "YEAR_TO_MONTH" | "MONTH" | "QUARTER" | "WEEK" | "DAY" | "DAY_TO_HOUR" | "DAY_TO_MINUTE" | "DAY_TO_SECOND" | "HOUR" | "HOUR_TO_MINUTE" | "HOUR_TO_SECOND" | "MINUTE" | "MINUTE_TO_SECOND" | "SECOND" ;

timePointUnit = "YEAR" | "MONTH" | "DAY" | "HOUR" | "MINUTE" | "SECOND" | "QUARTER" | "WEEK" | "MILLISECOND" | "MICROSECOND" ;

over = composite , "over" , fieldReference ;

overConstant = "current_row" | "current_range" | "unbounded_row" | "unbounded_row" ;

timeIndicator = fieldReference , "." , ( "proctime" | "rowtime" ) ;
```

**文字**：这里的文字是有效的Java文字。字符串文字可以使用单引号或双引号指定。复制引号以进行转义（例如“是我。”或“我”喜欢”狗。”）。

**空文字**：空文字必须附加一个类型。使用nullOf（type）（例如nullOf（INT））创建空值。

**字段引用**：fieldReference指定数据中的一列（如果使用*，则指定所有列），而functionIdentifier指定受支持的标量函数。列名和函数名遵循Java标识符语法。

**函数调用**：指定为字符串的表达式也可以使用前缀表示法而不是后缀表示法来调用运算符和函数。

**小数**：如果需要使用精确的数值或大的小数，则Table API还支持Java的BigDecimal类型。在Scala Table API中，小数可以由BigDecimal（“ 123456”）定义，而在Java中，可以通过附加“ p”来精确定义例如123456页

**时间表示**：为了使用时间值，Table API支持Java SQL的日期，时间和时间戳类型。在Scala Table API中，可以使用java.sql.Date.valueOf（“ 2016-06-27”），java.sql.Time.valueOf（“ 10:10:42”）或java.sql定义文字。Timestamp.valueOf（“ 2016-06-27 10：10：42.123”）。Java和Scala表API还支持调用“ 2016-06-27” .toDate（），“ 10:10:42” .toTime（）和“ 2016-06-27 10：10：42.123” .toTimestamp（）用于将字符串转换为时间类型。注意：由于Java的时态SQL类型取决于时区，因此请确保Flink Client和所有TaskManager使用相同的时区。

**时间间隔**：时间间隔可以表示为月数（Types.INTERVAL_MONTHS）或毫秒数（Types.INTERVAL_MILLIS）。可以添加或减去相同类型的间隔（例如1.小时+ 10分钟）。可以将毫秒间隔添加到时间点（例如“ 2016-08-10” .toDate + 5.days）。

**Scala表达式**:：Scala表达式使用隐式转换。因此，请确保将通配符导入org.apache.flink.table.api.scala._添加到程序中。如果文字不被视为表达式，请使用.toExpr（如3.toExpr）强制转换文字。