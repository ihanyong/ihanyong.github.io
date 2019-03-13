---
layout: post
title:  "【日常学习笔记】FlinkTableAPI的基本概念"
date:   2019-03-11 22:30:00 +0800
tags:
        - 流处理
        - java
        - Flink
---


# 概览
Flink 用TableAPI 和 SQL 来统一流处理与批处理。

## Setup
must:
```xml
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-table_2.11</artifactId>
  <version>1.7.2</version>
</dependency>
```

for batch query
```xml
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-scala_2.11</artifactId>
  <version>1.7.2</version>
</dependency>

```
for stream query
```xml
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-streaming-scala_2.11</artifactId>
  <version>1.7.2</version>
</dependency>
```

note: uber-jar里不要把flink-table打进去（ Apache Calcite 会阻止 classloader 的垃圾回收）。 可以让flink-talbe 放到 system 的 classloader 里 （将./opt/ 下的 flink-talbe.jar 拷贝到 ./lib/ 下）。

# 概念与通用API

## Tabel API 与 SQL 程序的结构
批处理和流处理对应的 Tabel API 与 SQL 程序的结构都是一样的。

```java
// 批处理将 StreamExecutionEnvironment 换为 ExecutionEnvironment 
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// 创建一个 TableEnvironment
// 批处理将 StreamTableEnvironment 换为 BatchTableEnvironment 
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// 注册一张表
tableEnv.registerTable("table1", ...)            // 或
tableEnv.registerTableSource("table2", ...);     // 或
tableEnv.registerExternalCatalog("extCat", ...);

// 注册一张输出表
tableEnv.registerTableSink("outputTable", ...);

// 通过 Table API 查询 创建一张表
Table tapiResult = tableEnv.scan("table1").select(...);
// 通过 SQL 查询 创建一张表
Table sqlResult  = tableEnv.sqlQuery("SELECT ... FROM table2 ... ");

// 将 Table API 的结果表发送到 TableSink,  SQL 结果的发送方式相同
tapiResult.insertInto("outputTable");

// execute
env.execute();
```

note: Table API 和 SQL 查询可以很容易的整合、嵌入到 DataStream 或 DataSet 程序中。 （DataStream\DataSet 可以与 Table 相互转换）

## 创建一个 TableEnvironment
TableEnvironment 是 Table API 和 SQL 的核心概念，用来:
- 注册 Table
- 注册 catalog
- 执行 SQL 查询
- 注册UDF（scalar, table, aggregation）
- 将 DataStream\DataSet 转换为 Table
- 保持对 ExecutionEnvironment\StreamExecutionEnvironment 的引用

一个 Table 永远绑定到一个特定 TableEnvironment。 不同TableEnvironment的表不能进行关联查询。

```java
// ***************
// 流查询
// ***************
StreamExecutionEnvironment sEnv = StreamExecutionEnvironment.getExecutionEnvironment();
// 为流查询创建一个 TableEnvironment 
StreamTableEnvironment sTableEnv = TableEnvironment.getTableEnvironment(sEnv);

// ***********
// 批查询
// ***********
ExecutionEnvironment bEnv = ExecutionEnvironment.getExecutionEnvironment();
// 为批查询创建一个 TableEnvironment 
BatchTableEnvironment bTableEnv = TableEnvironment.getTableEnvironment(bEnv);

```
## 表的注册
TableEnvironment 维护一份表的注册目录。 有两种表： 输入表和输出表。
输入表可以在 TableAPI\SQL 中被引用并提供输入数据。 输出表用来将TableAPI\SQL的查询结果发送到外部系统。

输入表可以通过多种源来注册
- 现有 table object。 一般是 TableAPI\SQL 的查询结果
- TableSource。 从外部获取数据，如文件，数据库，消息系统等。
- DataStream 或 DataSet。 从DataStream\DataSet 程序中获取

输出表可以通过 TableSink 来注册

### 现有表的注册
```java
// 获取 StreamTableEnvironment, （ BatchTableEnvironment 一样方式）
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// Table is the result of a simple projection query 
Table projTable = tableEnv.scan("X").select(...);

// register the Table projTable as table "projectedX"
tableEnv.registerTable("projectedTable", projTable);
```

### TableSource 的注册

```java
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// create a TableSource
TableSource csvSource = new CsvTableSource("/path/to/file", ...);

// register the TableSource as table "CsvTable"
tableEnv.registerTableSource("CsvTable", csvSource);
```

### TableSink 的注册

```java
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// create a TableSink
TableSink csvSink = new CsvTableSink("/path/to/file", ...);

// define the field names and types
String[] fieldNames = {"a", "b", "c"};
TypeInformation[] fieldTypes = {Types.INT, Types.STRING, Types.LONG};

// register the TableSink as table "CsvSinkTable"
tableEnv.registerTableSink("CsvSinkTable", fieldNames, fieldTypes, csvSink);
```

### 外部目录的注册
一个外部目录（external catalog）可以提供外部数据库\表的名字结构等信息及如何获取外部数据（数据库，表，文件等）.

外部目录可通过 ExternalCatalog  来定义， StreamTableEnvironment.registerExternalCatalog 注册
```java
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// create an external catalog
ExternalCatalog catalog = new InMemoryExternalCatalog();

// register the ExternalCatalog catalog
tableEnv.registerExternalCatalog("InMemCatalog", catalog);
```
外部目录一旦注册到了 TableEnvironment , 外部目录中定义的表就可以被 TableAPI\SQL 的查询语句通过指定全路径的方式来使用（如 catalog.database.table）。

当前 Flink 提供了 InMemoryExternalCatalog 作为 demo和测试用例。


## 查询 Table
### Table API
Table API 是基于Scala\Java 语法的查询API， 与SQL的区别在于， 不是通过字符串来指定查询，而是通过宿主语言的一步一步的组合调用来编排查询语法。

Table 是API的核心类，用来表示一个表，并提供了关系算子方法。 这些方法返回一个新的 Table对象，表示在输入表上应用算子后的结果。 一些关系算子是由若干方法调用组合来的如 _table.groupBy(...).select(...)_。

详细参考 [Table API 的文档](https://ci.apache.org/projects/flink/flink-docs-release-1.7/dev/table/tableApi.html)

一个简单的例子
```java
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// register Orders table

// scan registered Orders table
Table orders = tableEnv.scan("Orders");
// compute revenue for all customers from France
Table revenue = orders
  .filter("cCountry === 'FRANCE'")
  .groupBy("cID, cName")
  .select("cID, cName, revenue.sum AS revSum");

// emit or convert Table
// execute query
```


### SQL
Flink 的SQL 基于 [Apache Calcite](https://calcite.apache.org/)的， 通过字符串来编排查询。

详细参考 [SQL 的文档](https://ci.apache.org/projects/flink/flink-docs-release-1.7/dev/table/sql.html)
两个简单的例子
```java
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

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

```java
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// register "Orders" table
// register "RevenueFrance" output table

// compute revenue for all customers from France and emit to "RevenueFrance"
tableEnv.sqlUpdate(
    "INSERT INTO RevenueFrance " +
    "SELECT cID, cName, SUM(revenue) AS revSum " +
    "FROM Orders " +
    "WHERE cCountry = 'FRANCE' " +
    "GROUP BY cID, cName"
  );

// execute query
```

### Table API 与 SQL 混合使用
因为都是返回的 Table 对象， Table API and SQL 可以容易地混合在一起使用。

- 可以在 SQL查询返回的 Table 对象上执行 Table API 查询
- 可以在 Table API 查询返回的 Table 对象上执行 SQL查询

## 发送一张表

通过将表写入 TableSink 来发送一张表。 TableSink 是一个支持多种文件格式（CSV, Apache Parquet, Apache Avro...），存储系统（ JDBC, Apache HBase, Apache Cassandra, Elasticsearch...），和消息系统(Apache Kafka, RabbitMQ...)的接口

批处理的表只能写入 BatchTableSink， 而流表可以写入AppendStreamTableSink,  RetractStreamTableSink, 或 UpsertStreamTableSink。


关于可用的使用和自定义TableSink，参考 [Table Sources & Sinks](https://ci.apache.org/projects/flink/flink-docs-release-1.7/dev/table/sourceSinks.html) 

Table.insertInto(String tableName) 方法将表数据发送到一个注册了的TableSink。 这个方法会从 根据 name 从表目录中找到 TableSink， 并校验输出表的结构与 TableSink 的结构是一样的。

```java
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// create a TableSink
TableSink sink = new CsvTableSink("/path/to/file", fieldDelim = "|");

// register the TableSink with a specific schema
String[] fieldNames = {"a", "b", "c"};
TypeInformation[] fieldTypes = {Types.INT, Types.STRING, Types.LONG};
tableEnv.registerTableSink("CsvSinkTable", fieldNames, fieldTypes, sink);

// compute a result Table using Table API operators and/or SQL queries
Table result = ...
// emit the result Table to the registered TableSink
result.insertInto("CsvSinkTable");

// execute the program
```



## 翻译并执行查询

根据输入是流还是批，Table API 和 SQL 查询被翻译成 DataStream 或 DataSet 程序。 一个查询在内部以一个逻辑查询计划来表示，通过两步来翻译：
1. 优化逻辑计划
2. 翻译成 DataStream 或 DataSet 程序

翻译时机：
- 表被发送到 TableSink时， 如调用 Table.insertInto()
- 定义SQL 更新语句时，如调用 TableEnvironment.sqlUpdate()
- Table 转化为  DataStream 或 DataSet 时

一旦翻译完成，  Table API 或 SQL 查询就会像常规的 DataStream\DataSet 程序一样在调用StreamExecutionEnvironment.execute() 时 ExecutionEnvironment.execute()被处理执行了。

## 与 DataStream\DataSet 整合
Table API 和 SQL 查询 可以较容易地整合或嵌入到 DataStream\DataSet 程序中.比如 可以查询一个外部表 (如 RDBMS), 做一些预处理，如 过滤、投射、聚合 或连接一些元数据, 然后再用 DataStream\DataSet API进行一步对数据进行处理 (或者其它高级 APIs, 如 CEP、Gelly)。 反之, Table API 和 SQL 查询 也可以应用在DataStream\DataSet 程序的结果上。
可以通过将 DataStream\DataSet 转化为 Table 或反过来来实现这种交互。

### Scala 的隐式转换
The Scala Table API features implicit conversions for the DataSet, DataStream, and Table classes. These conversions are enabled by importing the package org.apache.flink.table.api.scala._ in addition to org.apache.flink.api.scala._ for the Scala DataStream API.

### 将 DataStream\DataSet 注册为表

DataStream\DataSet 可以在 TableEnvironment 中注册为一个表， 表的结构依赖于 DataStream\DataSet 的数据类型。 详见数据类型与表结果的映射一节。

```java
// get StreamTableEnvironment
// registration of a DataSet in a BatchTableEnvironment is equivalent
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

DataStream<Tuple2<Long, String>> stream = ...

// register the DataStream as Table "myTable" with fields "f0", "f1"
tableEnv.registerDataStream("myTable", stream);

// register the DataStream as table "myTable2" with fields "myLong", "myString"
tableEnv.registerDataStream("myTable2", stream, "myLong, myString")
```
note: 

Note: DataStream Table 的名字不能是 ^_DataStreamTable_[0-9]+ 模式，DataSet Table的名字不能是 ^_DataSetTable_[0-9]+ 模式。 这些模式是内部保留的


### 将 DataStream\DataSet  转换为表

除了将 DataStream\DataSet 注册为 TableEnvironment 的表，还可以直接将它们转换为一个表。 

```java
// get StreamTableEnvironment
// registration of a DataSet in a BatchTableEnvironment is equivalent
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

DataStream<Tuple2<Long, String>> stream = ...

// Convert the DataStream into a Table with default fields "f0", "f1"
Table table1 = tableEnv.fromDataStream(stream);

// Convert the DataStream into a Table with fields "myLong", "myString"
Table table2 = tableEnv.fromDataStream(stream, "myLong, myString");
```



### 将表转换为 DataStream\DataSet 
表可以转化为 DataStream\DataSet。这样自定义的 DataStream\DataSet 代码可以跑在 TableAPI\SQL 查询的结果上。

将表转化为 DataStream\DataSet时， 需要指定结果 DataStream\DataSet 的数据类型（表的行数据如何映射到数据类型上）。有如下几个选项（ 最方便使用的就是 Row 了）：
- Row: 通过位置映射字段，字段数量不限， 支持null， 非类型安全。
- POJO: 通过字段名映射字段（POJO字段名必须与 表字段名一致），字段数量不限， 支持null， 类型安全。
- Case Class: 通过位置映射字段，字段数量不限， 不支持null， 类型安全。
- Tuple: 通过位置映射字段，字段数量有限制(Scala 22 ， Java 25)， 不支持null， 类型安全。 
- Atomic Type: 只能有一个字段，不支持null, 类型安全

#### Convert a Table into a DataStream
流式查询结果的表会动态更新，即当新记录到达查询的输入流时，该表将发生更改。 因此，将这种动态查询转换为的数据流需要对表的更新进行编码。

表转为 DataStream 有两种模式:

- Append 模式: 只能用于 动态表有 INSERT 更新的场景，即，数据只能追加，前面的数据不会被更新。
- Retract 模式: 此模式可以用在任何情况下。 会用一个布尔型的标志位来标识是 INSERT 还是 DELETE 的更新。


```java
// get StreamTableEnvironment. 
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

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
note: Note: 详见 [动态表文档](https://ci.apache.org/projects/flink/flink-docs-release-1.7/dev/table/streaming/dynamic_tables.html) .

#### 将 Table 转换为 DataSet

```java
// get BatchTableEnvironment
BatchTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

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

### 数据类型与表结构的映射

Flink’s DataStream 和 DataSet APIs 支持很多种数据类型。 复合类型：Tuples， POJOs， Scala case classes, Flink 的 Row；其它的原子类型。 下面讨论 Table API 是如何转换并在内部表示将这些类型的。

数据类型的映射有两种方式: 基于位置的和基于字段名。


#### 基于位置的映射

基于位置的映射可以在保持字段顺序的同时赋予它们有意义的名称。 这种映射方式可以应用于有字段顺序的复合类型也可以用于原子类型。 像 tuples, rows 和 case classes 等复合类型是有字段顺序的。 而 POJO 没有字段顺序，必须使用基于字段名的映射方式。

使用基于位置的映射时， 指定的字段名不能与 输入数据类型的字段名冲突， 否则 API 会认为使用的是基于字段名的映射方式。
如果没有指定字段名， 对于复合类型会使用该类型的默认字段名与字段顺序，对于原子类型会使用 f0 作为字段名。

```java
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

DataStream<Tuple2<Long, Integer>> stream = ...

// 使用默认字段名 "f0"， "f1"， 将流转换为表
Table table = tableEnv.fromDataStream(stream);

// 使用字段名 "myLong"， "myInt"， 将流转换为表
Table table = tableEnv.fromDataStream(stream, "myLong, myInt");
```

#### Name-based Mapping

基于字段名的映射适用于所有的数据类型，是最灵活的定义表结构的方式。 所有的字段都是以名字来映射的， 也可以为其指定别名， 可以重排字段，或选取指定字段。

 如果没有指定字段名， 对于复合类型会使用该类型的默认字段名与字段顺序，对于原子类型会使用 f0 作为字段名。

```java
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

DataStream<Tuple2<Long, Integer>> stream = ...

// 使用默认字段名 "f0"， "f1"， 将流转换为表
Table table = tableEnv.fromDataStream(stream);

// 只将字段 "f1" 从流转换到表
Table table = tableEnv.fromDataStream(stream, "f1");

// 从流转换到表，并交换字段"f0"，"f1"的顺序
Table table = tableEnv.fromDataStream(stream, "f1, f0");

// 从流转换到表，并交换字段"f0"，"f1"的顺序，并命名字段为 "myInt", "myLong"
Table table = tableEnv.fromDataStream(stream, "f1 as myInt, f0 as myLong");
```

#### 原子类型
Flink 将原始类型 (Integer, Double, String) 或一般类型 (无法识别解析出来的类型) 看作是原子类型。 原子类型的 DataStream 或 DataSet 映射为只有一个字段的表。 字段类型会根据原子类型来推断， 字段名可以指定。

```java
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

DataStream<Long> stream = ...

// convert DataStream into Table with default field name "f0"
Table table = tableEnv.fromDataStream(stream);

// convert DataStream into Table with field name "myLong"
Table table = tableEnv.fromDataStream(stream, "myLong");
```
#### Tuples (Scala and Java) and Case Classes (Scala only)


Flink 支持 Scala 内置的 tuples 也为Java提供自己实现的 tuple 类。 tuples 类型的   DataStreams 和 DataSets 都可以转换为表。 可以通过为所有字段（基于位置）提供一个名字的方式来重命名字段。 如果没有提供名字，则使用默认字段名。 如果使用了原始的字段名（），则 API 会认为使用的是基于字段名的映射方式。 基于字段名映射可以重排字段顺序，也可以对字段进行投射（sql select as）

```java
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

DataStream<Tuple2<Long, String>> stream = ...

//以默认的字段名 "f0", "f1" 将流转为表
Table table = tableEnv.fromDataStream(stream);

// 以字段名 "myLong", "myString" (基于位置)  流转为表
Table table = tableEnv.fromDataStream(stream, "myLong, myString");

// 重排字段 "f1", "f0" (基于字段名) 流转表
Table table = tableEnv.fromDataStream(stream, "f1, f0");

// 只选择字段 "f1" (基于字段名) 进行流转表
Table table = tableEnv.fromDataStream(stream, "f1");

// 重排并重命名字段 "myString", "myLong" (基于字段名) 流转表
Table table = tableEnv.fromDataStream(stream, "f1 as 'myString', f0 as 'myLong'");
```
#### POJO (Java and Scala)

Flink 支持将 POJOs 作为复合类型。 POJO 的规则参照 [here](https://ci.apache.org/projects/flink/flink-docs-release-1.7/dev/api_concepts.html#pojos).

将 POJO DataStream 或 DataSet 转换为表时，不指定字段名时， 使用 POJO 的原始字段名作为表的字段名。 POJO 只能基于字段名来映射， 可以重命名（as）， 重排序， 投射。

```java
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// Person is a POJO with fields "name" and "age"
DataStream<Person> stream = ...

// convert DataStream into Table with default field names "age", "name" (fields are ordered by name!)
Table table = tableEnv.fromDataStream(stream);

// convert DataStream into Table with renamed fields "myAge", "myName" (name-based)
Table table = tableEnv.fromDataStream(stream, "age as myAge, name as myName");

// convert DataStream into Table with projected field "name" (name-based)
Table table = tableEnv.fromDataStream(stream, "name");

// convert DataStream into Table with projected and renamed field "myName" (name-based)
Table table = tableEnv.fromDataStream(stream, "name as myName");
```
#### Row
Row 数据类型支持任意数目的字段，和null 值。 字段名可以通过 RowTypeInfo 或者 DataStream  DataSet 转表时指定。 Row 类型支持基于字段名也支持基于位置的映射。 字段可以通过提供全部字段名（基于位置）来重命名， 或者选择部分的字段（基于字段名）来投射、重排、重命名。

```java
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// DataStream of Row with two fields "name" and "age" specified in `RowTypeInfo`
DataStream<Row> stream = ...

// convert DataStream into Table with default field names "name", "age"
Table table = tableEnv.fromDataStream(stream);

// convert DataStream into Table with renamed field names "myName", "myAge" (position-based)
Table table = tableEnv.fromDataStream(stream, "myName, myAge");

// convert DataStream into Table with renamed fields "myName", "myAge" (name-based)
Table table = tableEnv.fromDataStream(stream, "name as myName, age as myAge");

// convert DataStream into Table with projected field "name" (name-based)
Table table = tableEnv.fromDataStream(stream, "name");

// convert DataStream into Table with projected and renamed field "myName" (name-based)
Table table = tableEnv.fromDataStream(stream, "name as myName");
```

## 查询优化
Apache Flink 通过 Apache Calcite 优化、翻译 查询。 目前执行的优化包括投影和过滤器下推、子查询去关联和其他类型的查询重写。Flink还没有优化联接顺序，而是按照查询中定义的相同顺序（FROM子句中表的顺序和/或WHERE子句中联接谓词的顺序）执行它们。

通过提供 CalciteConfig 对象，可以调整在不同阶段应用的优化规则集。这可以通过调用 CalciteConfig.createBuilder() 生成器创建，并通过调用 tableEnv.getConfig.setCalciteConfig(calciteConfig) 提供给TableEnvironment。


### Explaining

TableAPI提供了一种机制来解释计算表的逻辑和优化查询计划。这是通过 TableEnvironment.explain(table) 方法完成的。它返回一个字符串，描述三个计划：

1. 关系查询的抽象语法树， 即未优化的逻辑查询计划
2. 优化的逻辑查询计划
3. 物理执行计划

代码如下：

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
StreamTableEnvironment tEnv = TableEnvironment.getTableEnvironment(env);

DataStream<Tuple2<Integer, String>> stream1 = env.fromElements(new Tuple2<>(1, "hello"));
DataStream<Tuple2<Integer, String>> stream2 = env.fromElements(new Tuple2<>(1, "hello"));

Table table1 = tEnv.fromDataStream(stream1, "count, word");
Table table2 = tEnv.fromDataStream(stream2, "count, word");
Table table = table1
  .where("LIKE(word, 'F%')")
  .unionAll(table2);

String explanation = tEnv.explain(table);
System.out.println(explanation);
```

```
== Abstract Syntax Tree ==
LogicalUnion(all=[true])
  LogicalFilter(condition=[LIKE($1, 'F%')])
    LogicalTableScan(table=[[_DataStreamTable_0]])
  LogicalTableScan(table=[[_DataStreamTable_1]])

== Optimized Logical Plan ==
DataStreamUnion(union=[count, word])
  DataStreamCalc(select=[count, word], where=[LIKE(word, 'F%')])
    DataStreamScan(table=[[_DataStreamTable_0]])
  DataStreamScan(table=[[_DataStreamTable_1]])

== Physical Execution Plan ==
Stage 1 : Data Source
  content : collect elements with CollectionInputFormat

Stage 2 : Data Source
  content : collect elements with CollectionInputFormat

  Stage 3 : Operator
    content : from: (count, word)
    ship_strategy : REBALANCE

    Stage 4 : Operator
      content : where: (LIKE(word, 'F%')), select: (count, word)
      ship_strategy : FORWARD

      Stage 5 : Operator
        content : from: (count, word)
        ship_strategy : REBALANCE
```

