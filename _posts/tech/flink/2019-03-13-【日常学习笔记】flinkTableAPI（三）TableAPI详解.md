---
layout: post
title:  "【日常学习笔记】flinkTableAPI（三）TableAPI详解"
date:   2019-03-13 22:30:00 +0800
tags:
        - 流处理
        - java
        - Flink
---

# 概览与示例

The Table API is available for Scala and Java. The Scala Table API leverages on Scala expressions, the Java Table API is based on strings which are parsed and converted into equivalent expressions.

The following example shows the differences between the Scala and Java Table API. The table program is executed in a batch environment. It scans the `Orders` table, groups by field `a`, and counts the resulting rows per group. The result of the table program is converted into a `DataSet` of type `Row` and printed.

Table API 可用于 Java 或 Scala。 Scala Table API 利用 Scala 表达式， Java Table API 则是基于可转换为等价表达式的字符串。

下面的示例展示了 Scala 与 Java Table API 的区别。 Table 程序在批环境中执行。 扫描 `Orders`表，根据字段 `a` 来分组统计记录的行数。 最后将结果转化为一个 `Row` 类型的  `DataSet`  并打印出来。

- Java
```java
// environment configuration
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
BatchTableEnvironment tEnv = TableEnvironment.getTableEnvironment(env);

// register Orders table in table environment
// ...

// specify table program
Table orders = tEnv.scan("Orders"); // schema (a, b, c, rowtime)

Table counts = orders
        .groupBy("a")
        .select("a, b.count as cnt");

// conversion to DataSet
DataSet<Row> result = tEnv.toDataSet(counts, Row.class);
result.print();
```

- Scala
```scala
import org.apache.flink.api.scala._
import org.apache.flink.table.api.scala._

// environment configuration
val env = ExecutionEnvironment.getExecutionEnvironment
val tEnv = TableEnvironment.getTableEnvironment(env)

// register Orders table in table environment
// ...

// specify table program
val orders = tEnv.scan("Orders") // schema (a, b, c, rowtime)

val result = orders
               .groupBy('a)
               .select('a, 'b.count as 'cnt)
               .toDataSet[Row] // conversion to DataSet
               .print()
              
```



下面一个例子展示了一个更复杂的 Table API 程序。 该程序也扫描了`Orders`表， 过滤掉null 值， 标准化字符串类型的字段 `a` ，然后以`a` 分组计算每小时字段`b`的平均数。 

```java
// environment configuration
// ...

// specify table program
Table orders = tEnv.scan("Orders"); // schema (a, b, c, rowtime)

Table result = orders
        .filter("a.isNotNull && b.isNotNull && c.isNotNull")
        .select("a.lowerCase() as a, b, rowtime")
        .window(Tumble.over("1.hour").on("rowtime").as("hourlyWindow"))
        .groupBy("hourlyWindow, a")
        .select("a, hourlyWindow.end as hour, b.avg as avgBillingAmount");
```

对于批和流 TableAPI 是统一的，所有上面两个例子可以不做修改地在批环境也可以在流环境中执行。不考虑流数据延迟的情况下， 在批与流中执行的结果是一样的。

# Operations

The Table API supports the following operations. Please note that not all operations are available in both batch and streaming yet; the are tagged accordingly(B:batch; S:streaming).

Table API 支持如下算子。 注意不是所有的算子都可以用同时用在批与流上的，会打上标签来说明(B:batch; S:streaming)。

## Scan, Projection, and Filter

Java :

| Operators    | tags | Description                                                  |
| ------------ | ---- | ------------------------------------------------------------ |
| Scan         | B,S  | 类似于SQL中的FROM语句。扫描一个注册过的表。<br/> <p>`  Table orders = tableEnv.scan("Orders");`</p> |
| Select       | B,S  | 类似于SQL中的SELECT语句。 执行select 操作。<br/><p>``` Table orders = tableEnv.scan("Orders"); Table result = orders.select("a, c as d"); ```</p><br/>可以使用通配符（*）来选取表中全部字段。<br/><p>```  Table result = orders.select("\*"); ``` </p> |
| As           | B,S  | 重命名字段<br/><p>Table orders = tableEnv.scan("Orders"); Table result = orders.as("x, y, z, t");</p> |
| Where/Filter | B,S  | 类似于SQL的Where。 过滤掉不符合条件的行。<br/><p>Table orders = tableEnv.scan("Orders"); Table result = orders.where("b === 'red'");</p><br/>or<p>Table orders = tableEnv.scan("Orders"); Table result = orders.filter("a % 2 === 0");</p> |

## Aggreagetion

| Operators                  | tags | Description |
| -------------------------- | ---- | ----------- |
| GroupBy Aggregation        |      |             |
| GroupBy Window Aggregation |      |             |
| OverWindow Aggregation     |      |             |
| Distinct Aggregation       |      |             |
| Distinct                   |      |             |

## Joins

| Operators                           | tags                | Description                                                  |
| ----------------------------------- | ------------------- | ------------------------------------------------------------ |
| Inner Join                          | B,S                 | 类似于SQL中的 JOIN。 连接两张表。两张表的字段名不能有相同的，必须通过 join 算子使用where或filter定义至少有一个等值连接条件谓词<br/><p>Table left = tableEnv.fromDataSet(ds1, "a, b, c"); <br/>Table right = tableEnv.fromDataSet(ds2, "d, e, f"); <br/>Table result = left.join(right).where("a = d").select("a, b, e");</p><br/>note: 对于流查询用来计算查询结果的状态大小会随着不同输入数据增多而无限增长。请提供一个有效保留间隔的查询配置来防止状态存储耗尽。 |
| Outer Join                          | B,S,Result Updating | 类似SQL中的LEFT/RIGHT/FULL OUTER JOIN。连接两张表。两张表的字段名不能有相同的，至少有一个等值连接条件谓词<br/><p>Table left = tableEnv.fromDataSet(ds1, "a, b, c"); <br/>Table right = tableEnv.fromDataSet(ds2, "d, e, f");  <br/>Table leftOuterResult = left.leftOuterJoin(right, "a = d").select("a, b, e"); <br/>Table rightOuterResult = left.rightOuterJoin(right, "a = d").select("a, b, e"); <br/>Table fullOuterResult = left.fullOuterJoin(right, "a = d").select("a, b, e");</p><br/>note: 对于流查询用来计算查询结果的状态大小会随着不同输入数据增多而无限增长。请提供一个有效保留间隔的查询配置来防止状态存储耗尽。 |
| Time-windowed Join                  | B,S                 | Note: Time-Windowed Joins 可以流的方式处理常规连接的一个子集<br/>时间窗口连接要求至少一个等值连接的谓词，和一个限定连接两端的时间的连接条件。这个条件可以通过两个范围条件谓词(<,<=,>=,>)，或是一个等值条件谓词来比较两张表的相同类型的时间属性(基于处理时间或事件时间)<br/>如下面两个条件是有效的窗口连接条件：<br/><p>ltime === rtime
<br/>ltime >= rtime && ltime < rtime + 10.minutes</p><br/><p>Table left = tableEnv.fromDataSet(ds1, "a, b, c, ltime.rowtime");
<br/>Table right = tableEnv.fromDataSet(ds2, "d, e, f, rtime.rowtime");


Table result = left.join(right)

  .where("a = d && ltime >= rtime - 5.minutes && ltime < rtime + 10.minutes")

  .select("a, b, e, ltime");</p> |
| Inner Join with Table Function      | B,S                 | <br/><p></p>                                                 |
| Left Outer Join with Table Function | B,S                 | <br/><p></p>                                                 |
| Join with Temporal Table            | B,S                 | <br/><p></p>                                                 |

## Set Operations

| Operators    | tags | Description                                                  |
| ------------ | ---- | ------------------------------------------------------------ |
| Union        | B    | 类似SQL中的UNION。合并两张表，并去重。两张表的字段类型必须完全一致<br/><p>Table left = tableEnv.fromDataSet(ds1, "a, b, c"); <br/>Table right = tableEnv.fromDataSet(ds2, "a, b, c"); <br/>Table result = left.union(right);</p> |
| UnionAll     | B,S  | 类似SQL中的UNION ALL。合并两张表。两张表的字段类型必须完全一致 <br/><p>Table left = tableEnv.fromDataSet(ds1, "a, b, c"); <br/>Table right = tableEnv.fromDataSet(ds2, "a, b, c"); <br/>Table result = left.unionAll(right);</p> |
| Intersect    | B    | 类似SQL中的INTERSECT。 返回两张表的交集。如果某个记录在一张或两张表中出现多次，也只返回一条记录，即结果是去重的。两张表的字段类型必须完全一致。<br/><p>Table left = tableEnv.fromDataSet(ds1, "a, b, c"); <br/>Table right = tableEnv.fromDataSet(ds2, "d, e, f"); <br/>Table result = left.intersect(right);</p> |
| IntersectAll | B    | 类似SQL中的INTERSECT ALL。 返回两张表的交集。如果某个记录在两张表中出现多次，也会返回多条记录，即结果可能会有重复数据。两张表的字段类型必须完全一致<br/><p>Table left = tableEnv.fromDataSet(ds1, "a, b, c"); <br/>Table right = tableEnv.fromDataSet(ds2, "d, e, f"); <br/>Table result = left.intersectAll(right);</p> |
| Minus        | B    | 类似SQL中的EXCEPT。 返回两张表的差集（left-right）。如果某个记录在左表中出现多次，只返回一条记录，即结果去重。两张表的字段类型必须完全一致<br/><p>Table left = tableEnv.fromDataSet(ds1, "a, b, c"); <br/>Table right = tableEnv.fromDataSet(ds2, "a, b, c"); <br/>Table result = left.minus(right);</p> |
| MinusAll     | B    | 类似SQL中的EXCEPT ALL。 返回两张表的差集（left-right）。如果某个记录在左表中出现n次，在右表出现m次，只返回（n-m）条该记录，即从左表中按记录在右表出现过次数删除记录。两张表的字段类型必须完全一致<br/><p>Table left = tableEnv.fromDataSet(ds1, "a, b, c"); <br/>Table right = tableEnv.fromDataSet(ds2, "a, b, c"); <br/>Table result = left.minusAll(right);</p> |
| In           | B,S  | 类似于SQL的IN。如果一个表达式在子查询中存在则返回true。 子查询结果必须有且只有一个结果字段。结果字段的类型必须与表达式中的字段类型一致。<br/><p>Table left = ds1.toTable(tableEnv, "a, b, c"); <br/>Table right = ds2.toTable(tableEnv, "a");  <br/><br/>// using implicit registration <br/>Table result = left.select("a, b, c").where("a.in(" + right + ")");  <br/><br/>// using explicit registration tableEnv.registerTable("RightTable", right); <br/>Table result = left.select("a, b, c").where("a.in(RightTable)");</p><br/> |

## OrderBy, Offset & Fetch

| Operators      | tags | Description                                                  |
| -------------- | ---- | ------------------------------------------------------------ |
| Order By       | B    | 类似SQL中的 ORDER BY。 返回全局(所有的并行度分片)排序过的记录<br/><p>Table in = tableEnv.fromDataSet(ds, "a, b, c"); <br/>Table result = in.orderBy("a.asc");</p> |
| Offset & Fetch | B    | 类似SQL中的 OFFSET 和FETCH。 Offset和Fetch从排序结果中提取指定数量的结果行。在技术实现上Offset和Fetch是 OrderBy算子的一部分，所以必须要接在OrderBy后面。  <br/><p>Table in = tableEnv.fromDataSet(ds, "a, b, c"); <br/><br/> // returns the first 5 records from the sorted result <br/>Table result1 = in.orderBy("a.asc").fetch(5);   <br/><br/>// skips the first 3 records and returns all following records from the sorted result <br/>Table result2 = in.orderBy("a.asc").offset(3);  <br/><br/>// skips the first 10 records and returns the next 5 records from the sorted result <br/>Table result3 = in.orderBy("a.asc").offset(10).fetch(5);</p> |

## Insert

| Operators   | tags | Descirption                                                  |
| ----------- | ---- | ------------------------------------------------------------ |
| Insert Into | B,S  | 类似SQL中的 INSERT INTO 。执行insert into 操作到一个注册的输出表<br/>输出表必须在 TableEnvironment 中注册过。 注册的表的模式要与查询的模式相匹配<br/><p>Table orders = tableEnv.scan("Orders");<br> orders.insertInto("OutOrders");</p> |

