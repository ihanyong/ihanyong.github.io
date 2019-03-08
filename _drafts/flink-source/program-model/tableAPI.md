TAbleAPI_and_SQL

# 概览
Flink 用 tableAPI t SQL 来统一流处理与批处理。

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
- Retract 模式: This mode can always be used. It encodes INSERT and DELETE changes with a boolean flag.

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
note: Note: A detailed discussion about dynamic tables and their properties is given in the [Dynamic Tables](https://ci.apache.org/projects/flink/flink-docs-release-1.7/dev/table/streaming/dynamic_tables.html) document.

#### Convert a Table into a DataSet
A Table is converted into a DataSet as follows:

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
### 
## 查询优化
### Explaining
