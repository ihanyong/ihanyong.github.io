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

Table API 支持如下算子。 注意不是所有的算子都可以用同时用在批与流上的，会打上标签来说明(B:batch; S:streaming)。

## Scan, Projection, and Filter

Java :

| Operators    | tags | Description                                                  |
| ------------ | ---- | ------------------------------------------------------------ |
| Scan         | B,S  | 类似于SQL中的FROM语句。扫描一个注册过的表。<br/>  ``` Table orders = tableEnv.scan("Orders");``` |
| Select       | B,S  | 类似于SQL中的SELECT语句。 执行select 操作。<br/>```Table orders = tableEnv.scan("Orders"); Table result = orders.select("a, c as d"); ```<br/>可以使用通配符（_*_）来选取表中全部字段。<br/>  Table result = orders.select("_*_");  |
| As           | B,S  | 重命名字段<br/>```Table orders = tableEnv.scan("Orders"); Table result = orders.as("x, y, z, t");``` |
| Where/Filter | B,S  | 类似于SQL的Where。 过滤掉不符合条件的行。<br/>```Table orders = tableEnv.scan("Orders");<br/> Table result = orders.where("b === 'red'");```<br/>```orTable orders = tableEnv.scan("Orders"); Table result = orders.filter("a % 2 === 0");``` |

## Aggreagetion

| Operators                  | tags                | Description                                                  |
| -------------------------- | ------------------- | ------------------------------------------------------------ |
| GroupBy Aggregation        | B,S,Result Updating | 类似于SQL的 GROUP BY。 根据分组key 对行进行分组，并接一个聚合算子来对分组的行进行聚合运算。 <br/>```Table orders = tableEnv.scan("Orders");```<br/>```Table result = orders.groupBy("a").select("a, b.sum as d");```<br/> <br/> Note: 对于流查询用来计算查询结果的状态大小会随着不同输入数据增多而无限增长。请提供一个有效保留间隔的查询配置来防止状态存储耗尽。 |
| GroupBy Window Aggregation | B,S                 | 在分组窗口上进行分组聚合一个表。 可能是一个或多个分组key <br/>``` Table orders = tableEnv.scan("Orders");``` <br/>```Table result = orders .window(Tumble.over("5.minutes").on("rowtime").as("w")) // define ```<br/>```window.groupBy("a, w") // group by key and``` <br/>```window.select("a, w.start, w.end, w.rowtime, b.sum as d"); // access window properties and aggregate``` ​ |
| OverWindow Aggregation     | S                   | 类似于SQL 的 OVER。 基于窗口范围内的前后行，对每行进行计算<br/> <br/>```Table orders = tableEnv.scan("Orders"); Table result = orders <br/>// define <br/>window .window(Over<br/> .partitionBy("a")<br/> .orderBy("rowtime") <br/>.preceding("UNBOUNDED_RANGE")<br/> .following("CURRENT_RANGE")<br/> .as("w")) <br/>.select("a, b.avg over w, b.max over w, b.min over w"); // sliding aggregate ```<br/>Note: 所有的聚合必须在相同的窗口上定义，即相同的分片，排序和范围。 当前只支持的之前到现在数据范围的窗口。 还不支持后续范围。 必须在单独地时间属性上指定ORDER BY。 |
| Distinct Aggregation       | B,S,Result Updating | 与SQL的 DISTINCT聚合类似，如 COUNT(DISTINCT a)。 Distinct aggregation 声明了聚合函数（内置或自定义）只应用于distinct 的输入值上。Distinct 可以用在 GroupBy Aggregation, GroupBy Window Aggregation 和 Over Window Aggregation. <br/>  ```Table orders = tableEnv.scan("Orders"); // Distinct aggregation on group by Table groupByDistinctResult = orders .groupBy("a") .select("a, b.sum.distinct as d"); // Distinct aggregation on time window group by Table groupByWindowDistinctResult = orders .window(Tumble.over("5.minutes").on("rowtime").as("w")).groupBy("a, w") .select("a, b.sum.distinct as d");<br/> // Distinct aggregation on over window <br/>Table result = orders .window(Over .partitionBy("a") .orderBy("rowtime") .preceding("UNBOUNDED_RANGE") .as("w")) .select("a, b.avg.distinct over w, b.max over w, b.min over w"); ```<br/> 自定义的聚合函数也可以使用DISTINCT修饰符。 想要只针对distinct 值做聚合，只需要在聚合函数后加上distinct修饰符。  <br/>```Table orders = tEnv.scan("Orders"); <br/>// Use distinct aggregation for user-defined aggregate functions <br/>tEnv.registerFunction("myUdagg", new MyUdagg()); <br/>orders.groupBy("users").select("users, myUdagg.distinct(points) as myDistinctResult");```<br/> Note: 对于流查询用来计算查询结果的状态大小会随着不同输入数据增多而无限增长。请提供一个有效保留间隔的查询配置来防止状态存储耗尽。 |
| Distinct                   | B,S,Result Updating | 与SQL 的DISTINCT 类似。 返回去重的结果记录 <br/> ``` Table orders = tableEnv.scan("Orders"); <br/>Table result = orders.distinct(); ```<br/> Note: 对于流查询用来计算查询结果的状态大小会随着不同输入数据增多而无限增长。请提供一个有效保留间隔的查询配置来防止状态存储耗尽。 |

## Joins

| Operators                           | tags                | Description                                                  |
| ----------------------------------- | ------------------- | ------------------------------------------------------------ |
| Inner Join                          | B,S                 | 类似于SQL中的 JOIN。 连接两张表。两张表的字段名不能有相同的，必须通过 join 算子使用where或filter定义至少有一个等值连接条件谓词<br/>```Table left = tableEnv.fromDataSet(ds1, "a, b, c"); <br/>Table right = tableEnv.fromDataSet(ds2, "d, e, f"); <br/>Table result = left.join(right).where("a = d").select("a, b, e");```<br/>note: 对于流查询用来计算查询结果的状态大小会随着不同输入数据增多而无限增长。请提供一个有效保留间隔的查询配置来防止状态存储耗尽。 |
| Outer Join                          | B,S,Result Updating | 类似SQL中的LEFT/RIGHT/FULL OUTER JOIN。连接两张表。两张表的字段名不能有相同的，至少有一个等值连接条件谓词<br/>```Table left = tableEnv.fromDataSet(ds1, "a, b, c"); <br/>Table right = tableEnv.fromDataSet(ds2, "d, e, f");  <br/>Table leftOuterResult = left.leftOuterJoin(right, "a = d").select("a, b, e"); <br/>Table rightOuterResult = left.rightOuterJoin(right, "a = d").select("a, b, e"); <br/>Table fullOuterResult = left.fullOuterJoin(right, "a = d").select("a, b, e");```<br/>note: 对于流查询用来计算查询结果的状态大小会随着不同输入数据增多而无限增长。请提供一个有效保留间隔的查询配置来防止状态存储耗尽。 |
| Time-windowed Join                  | B,S                 | Note: Time-Windowed Joins 可以流的方式处理常规连接的一个子集<br/>时间窗口连接要求至少一个等值连接的谓词，和一个限定连接两端的时间的连接条件。这个条件可以通过两个范围条件谓词(<,<=,>=,>)，或是一个等值条件谓词来比较两张表的相同类型的时间属性(基于处理时间或事件时间)<br/>如下面两个条件是有效的窗口连接条件：<br/>`ltime === rtime` <br/>`ltime >= rtime && ltime < rtime + 10.minutes`<br/>```Table left = tableEnv.fromDataSet(ds1, "a, b, c, ltime.rowtime"); <br/>Table right = tableEnv.fromDataSet(ds2, "d, e, f, rtime.rowtime"); Table result = left.join(right) .where("a = d && ltime >= rtime - 5.minutes && ltime < rtime + 10.minutes") .select("a, b, e, ltime"); ```|
| Inner Join with Table Function      | B,S                 | 根据表函数的结果来连接表。左表的每条记录被连接到调用的表函数产生的全部记录。 对于左表的一条记录，如果用他调用表函数返回的结果是空集，则该记录被丢弃。 <br/> ``` // register User-Defined Table Function TableFunction<String> split = new MySplitUDTF(); tableEnv.registerFunction("split", split); // join Table orders = tableEnv.scan("Orders"); Table result = orders .join(new Table(tableEnv, "split(c)").as("s", "t", "v")) .select("a, b, s, t, v"); ​``` |
| Left Outer Join with Table Function | B,S                 | 根据表函数的结果来连接表。左表的每条记录被连接到调用的表函数产生的全部记录。 对于左表的一条记录，如果用他调用表函数返回的结果是空集，则该记录会被保留，并填充null值。 <br/> Note: 当前，表函数左外部联接的谓词结果只能为空或字符串"true"。<br/> ``` // register User-Defined Table Function TableFunction<String> split = new MySplitUDTF(); tableEnv.registerFunction("split", split); // join Table orders = tableEnv.scan("Orders"); Table result = orders .leftOuterJoin(new Table(tableEnv, "split(c)").as("s", "t", "v")) .select("a, b, s, t, v"); ``` |
| Join with Temporal Table            | B,S                 | 历史表是随着时间追踪的变更的表。 <br/><br/>历史表函数允许访问在指定时间点上历史表的状态。 连接历史表的语法与内连接一个表函数是一样的。 当前历史表只支持内连接。 ```java Table ratesHistory = tableEnv.scan("RatesHistory"); // register temporal table function with a time attribute and primary key TemporalTableFunction rates = ratesHistory.createTemporalTableFunction("r_proctime", "r_currency"); tableEnv.registerFunction("rates", rates); // join with "Orders" based on the time attribute and key Table orders = tableEnv.scan("Orders"); Table result = orders .join(new Table(tEnv, "rates(o_proctime)"), "o_currency = r_currency") ``` |

## Set Operations

| Operators    | tags | Description                                                  |
| ------------ | ---- | ------------------------------------------------------------ |
| Union        | B    | 类似SQL中的UNION。合并两张表，并去重。两张表的字段类型必须完全一致<br/>```Table left = tableEnv.fromDataSet(ds1, "a, b, c"); <br/>Table right = tableEnv.fromDataSet(ds2, "a, b, c"); <br/>Table result = left.union(right);``` |
| UnionAll     | B,S  | 类似SQL中的UNION ALL。合并两张表。两张表的字段类型必须完全一致 <br/>```Table left = tableEnv.fromDataSet(ds1, "a, b, c"); <br/>Table right = tableEnv.fromDataSet(ds2, "a, b, c"); <br/>Table result = left.unionAll(right); ```|
| Intersect    | B    | 类似SQL中的INTERSECT。 返回两张表的交集。如果某个记录在一张或两张表中出现多次，也只返回一条记录，即结果是去重的。两张表的字段类型必须完全一致。<br/>```Table left = tableEnv.fromDataSet(ds1, "a, b, c"); <br/>Table right = tableEnv.fromDataSet(ds2, "d, e, f"); <br/>Table result = left.intersect(right); ```|
| IntersectAll | B    | 类似SQL中的INTERSECT ALL。 返回两张表的交集。如果某个记录在两张表中出现多次，也会返回多条记录，即结果可能会有重复数据。两张表的字段类型必须完全一致<br/>```Table left = tableEnv.fromDataSet(ds1, "a, b, c"); <br/>Table right = tableEnv.fromDataSet(ds2, "d, e, f"); <br/>Table result = left.intersectAll(right); ```|
| Minus        | B    | 类似SQL中的EXCEPT。 返回两张表的差集（left-right）。如果某个记录在左表中出现多次，只返回一条记录，即结果去重。两张表的字段类型必须完全一致<br/>```Table left = tableEnv.fromDataSet(ds1, "a, b, c"); <br/>Table right = tableEnv.fromDataSet(ds2, "a, b, c"); <br/>Table result = left.minus(right); ```|
| MinusAll     | B    | 类似SQL中的EXCEPT ALL。 返回两张表的差集（left-right）。如果某个记录在左表中出现n次，在右表出现m次，只返回（n-m）条该记录，即从左表中按记录在右表出现过次数删除记录。两张表的字段类型必须完全一致<br/>```Table left = tableEnv.fromDataSet(ds1, "a, b, c"); <br/>Table right = tableEnv.fromDataSet(ds2, "a, b, c"); <br/>Table result = left.minusAll(right);``` |
| In           | B,S  | 类似于SQL的IN。如果一个表达式在子查询中存在则返回true。 子查询结果必须有且只有一个结果字段。结果字段的类型必须与表达式中的字段类型一致。<br/>```Table left = ds1.toTable(tableEnv, "a, b, c"); <br/>Table right = ds2.toTable(tableEnv, "a");  <br/><br/>// using implicit registration <br/>Table result = left.select("a, b, c").where("a.in(" + right + ")");  <br/><br/>// using explicit registration tableEnv.registerTable("RightTable", right); <br/>Table result = left.select("a, b, c").where("a.in(RightTable)");<br/> ```|

## OrderBy, Offset & Fetch

| Operators      | tags | Description                                                  |
| -------------- | ---- | ------------------------------------------------------------ |
| Order By       | B    | 类似SQL中的 ORDER BY。 返回全局(所有的并行度分片)排序过的记录<br/>```Table in = tableEnv.fromDataSet(ds, "a, b, c"); <br/>Table result = in.orderBy("a.asc"); ```|
| Offset & Fetch | B    | 类似SQL中的 OFFSET 和FETCH。 Offset和Fetch从排序结果中提取指定数量的结果行。在技术实现上Offset和Fetch是 OrderBy算子的一部分，所以必须要接在OrderBy后面。  <br/>```Table in = tableEnv.fromDataSet(ds, "a, b, c"); <br/><br/> // returns the first 5 records from the sorted result <br/>Table result1 = in.orderBy("a.asc").fetch(5);   <br/><br/>// skips the first 3 records and returns all following records from the sorted result <br/>Table result2 = in.orderBy("a.asc").offset(3);  <br/><br/>// skips the first 10 records and returns the next 5 records from the sorted result <br/>Table result3 = in.orderBy("a.asc").offset(10).fetch(5); ```|

## Insert

| Operators   | tags | Descirption                                                  |
| ----------- | ---- | ------------------------------------------------------------ |
| Insert Into | B,S  | 类似SQL中的 INSERT INTO 。执行insert into 操作到一个注册的输出表<br/>输出表必须在 TableEnvironment 中注册过。 注册的表的模式要与查询的模式相匹配<br/>```Table orders = tableEnv.scan("Orders");<br> orders.insertInto("OutOrders"); ```|



## Group Window
组窗口根据时间或行计数间隔将组中的行聚合为有限组，并对每个组计算一次聚合函数。对于批处理表，Windows是按时间间隔对记录分组的便捷快捷方式。

窗口通过 window 算子来定义，并需要声明一个别名，可以在后面的语句中使用。 为了使用窗口来对表进行分组， 必须像常规使用 groupBy()那样使用将窗口别名做为分组key。 下面的代码展示了如何在表上进行窗口分组：
Java

```java
Table table = input
  .window([Window w].as("w"))  // define window with alias w
  .groupBy("w")  // group the table by window w
  .select("b.sum");  // aggregate
```
在流环境中， 如果通过groupBy(...) 指定除了窗口外还有其它的分组条件，窗口聚合只能进行并行计算。 一个只指定了窗口别名（一个）的groupBy(...) 语句会在一个单独的，非并行的任务中执行运算。 下面的代码展示了指定额外分组key 的情况。 

Java
```java
Table table = input
  .window([Window w].as("w"))  // define window with alias w
  .groupBy("w, a")  // group the table by attribute a and window w 
  .select("a, b.sum");  // aggregate
```

窗口的属性如开始时间，结束时间或行时间戳等可以在 select 的语句中通过 【w.start, w.end, and w.rowtime】 来使用。 窗口的开始时间与行时间是包含在窗口内的， 结束时间则不在窗口内。 例如： 一个从14:00开始的30分钟的的滚动窗口， 开始时间是【14:00:00.000】， 行时间是【14:29:59.999】， 结束时间是【14:30:00.000】


Java
```java
Table table = input
  .window([Window w].as("w"))  // define window with alias w
  .groupBy("w, a")  // group the table by attribute a and window w 
  .select("a, w.start, w.end, w.rowtime, b.count"); // aggregate and add window start, end, and rowtime timestamps
```

The Window parameter defines how rows are mapped to windows. Window is not an interface that users can implement. Instead, the Table API provides a set of predefined Window classes with specific semantics, which are translated into underlying DataStream or DataSet operations. The supported window definitions are listed below.

Window 参数定义了行是如何映射到窗口的， 用户不能自己实现Window接口。 Table API 提供了一系列不同语义的内置窗口类， 可以翻译为底层的 DataStream 或 DataSet 的算子。 下面是支持的窗口定义：


### Tumble (Tumbling Windows)
滚动窗口将行分配为没有重叠，固定长度的连续窗口。 如一个5分钟的滚动窗口将行分为5分钟一组。 滚动窗口可以基于事件时间，处理时间或行数来定义。

Tumbling windows are defined by using the Tumble class as follows:
滚动窗口通过 Tumble 类来定义：

| Method |  Description |
| ------ | ------------ |
| over | 定义窗口的长度，时间或行数间隔 |
| on   | 分组的时间属性（时间间隔）或行数。对于批查询可以是任何的 Long值或时间属性。对于流查询，必须是一个声明了的事件时间或处理时间属性。 |
| as   | 为窗口声名一个别名。 这个别名可以在后面的 groupBy()中使用， 也可以在select中用于获取窗口的属性（开始结束行时间）|

```java
// Tumbling Event-time Window
.window(Tumble.over("10.minutes").on("rowtime").as("w"));

// Tumbling Processing-time Window (assuming a processing-time attribute "proctime")
.window(Tumble.over("10.minutes").on("proctime").as("w"));

// Tumbling Row-count Window (assuming a processing-time attribute "proctime")
.window(Tumble.over("10.rows").on("proctime").as("w"));
```


### 滑动窗口
A sliding window has a fixed size and slides by a specified slide interval. If the slide interval is smaller than the window size, sliding windows are overlapping. Thus, rows can be assigned to multiple windows. For example, a sliding window of 15 minutes size and 5 minute slide interval assigns each row to 3 different windows of 15 minute size, which are evaluated in an interval of 5 minutes. Sliding windows can be defined on event-time, processing-time, or on a row-count.


滑动窗口大小固定并按指定的滑动步长进行滑动。 如果滑动步长小于窗口大小， 滑动窗口就会有重叠。 这样数据就会被分配给多个窗口。 如滑动步长为5分钟，大小了15分钟的滑动窗口，一行会被分配给3个窗口。滑动窗口可以基于事件时间，处理时间，行数来定义。

滑动窗口通过 Slide 类来定义：

| Method |  Description |
| ------ | ------------ |
| over | 定义窗口的长度，时间或行数间隔 |
| every | 定义窗口的步长，时间或行数间隔。步长必须与长度的类型一致 |
|on | 分组的时间属性（时间间隔）或行数。对于批查询可以是任何的 Long值或时间属性。对于流查询，必须是一个声明了的事件时间或处理时间属性。 |
| as | 为窗口声名一个别名。 这个别名可以在后面的 groupBy()中使用， 也可以在select中用于获取窗口的属性（开始结束行时间） |

```java
// Sliding Event-time Window
.window(Slide.over("10.minutes").every("5.minutes").on("rowtime").as("w"));

// Sliding Processing-time window (assuming a processing-time attribute "proctime")
.window(Slide.over("10.minutes").every("5.minutes").on("proctime").as("w"));

// Sliding Row-count window (assuming a processing-time attribute "proctime")
.window(Slide.over("10.rows").every("5.rows").on("proctime").as("w"));
```

### Session Window

Session windows do not have a fixed size but their bounds are defined by an interval of inactivity, i.e., a session window is closes if no event appears for a defined gap period. For example a session window with a 30 minute gap starts when a row is observed after 30 minutes inactivity (otherwise the row would be added to an existing window) and is closed if no row is added within 30 minutes. Session windows can work on event-time or processing-time.

会话窗口没有固定的大小， 而是由非活跃的间隔来决定，即超过指定时间没有事件出现的话，窗口就会结束。 如一个30分钟间隔的会话窗口， 在30分钟非活跃状态后，一条记录进来行会开始一个新窗口（没有超过30分钟的话数据会分配到当前窗口），如果30分钟内没有数据进来，窗口就会结束。 会话窗口可以基于事件时间或处理时间来定义。

会话窗口通过 Session 类来定义：

| Method |  Description |
| ------ | ------------ |
| withGap | 定义两个时间窗口之间的时间间隙 |
|on | 分组的时间属性（时间间隔）或行数。对于批查询可以是任何的 Long值或时间属性。对于流查询，必须是一个声明了的事件时间或处理时间属性。 |
| as | 为窗口声名一个别名。 这个别名可以在后面的 groupBy()中使用， 也可以在select中用于获取窗口的属性（开始结束行时间） |

## Over Window
Over window 聚合类似于标准SQL中OVER语句。 在查询的SELECT语句中的定义。 与通过 GROUP BY 语句指定的 group window 不同， over window不对行进行折叠， 而是针对每个输入行在它相邻范围的行上进行聚合计算。

Over windows 通过 window(w: OverWindow*) 算子定义，并在 select()方法中通过别名引用。

```java
Table table = input
  .window([OverWindow w].as("w"))           // define over window with alias w
  .select("a, b.sum over w, c.min over w"); // aggregate over the over window w
```

OverWindow 定义计算聚合的行的范围。 不能自定义OverWindow接口的实现。 Table API 提供了 Over 类来配置  over window 的属性。  over window 可以基于事件时间或处理时间定义一个时间间隔或行数的范围。 Over 类的方法如下：
---
#### partitionBy 可选
基于一个或多个属性对输入进行分片。每个分片都是独立地排序，并分别应用聚合函数。
Note： 在流环境中， 指定了partition by语句的话，窗口聚合会并行地进行计算。 没有指定 partition by 时，流是被一个单独的，非并行的任务处理。


#### orderBy 必须
定义每个分片内的排序，从而聚合函数以这个顺序应用到行上面。

Note： 对于流查询，必须 声明事件时间或处理时间属性。 当前只支持一个排序属性。


#### preceding 必须
定义当前行之前的包含在窗口内的行的间距。 间距可以被定义为时间也可以是行数。
- 有界的 over windows 通过间距大小来指定，如时间间距【10.minutes】， 行数间距【10.rows】
- 无界的 over windows 通过固定字符串来指定，如【UNBOUNDED_RANGE】来定义时间，【UNBOUNDED_ROW】来定义行数。 无界的 over windows 从该分片的第一条数据开始。

#### following 可选

定义当前行之后的包含在窗口内的行的间距。 间距可以被定义为时间也可以是行数。 必须与preceding定义使用的单位相同。

当前还不支持指定后续行。 可以指定如下两个常量字符串：

- CURRENT_ROW 将窗口上界设置为当前行。
- CURRENT_RANGE 将窗口上界设置为当前行的排序键，即：与当前行的排序键相同的所有行都包含在窗口内。

如果省略了 following， 时间窗口的上界默认为CURRENT_RANGE， 行数窗口的上界默认为CURRENT_ROW

##### as  必须
为窗口声名一个别名。 这个别名在select()中用来引用 over window。

---

### 无界的 Over window
```java
// Unbounded Event-time over window (assuming an event-time attribute "rowtime")
.window(Over.partitionBy("a").orderBy("rowtime").preceding("unbounded_range").as("w"));

// Unbounded Processing-time over window (assuming a processing-time attribute "proctime")
.window(Over.partitionBy("a").orderBy("proctime").preceding("unbounded_range").as("w"));

// Unbounded Event-time Row-count over window (assuming an event-time attribute "rowtime")
.window(Over.partitionBy("a").orderBy("rowtime").preceding("unbounded_row").as("w"));
 
// Unbounded Processing-time Row-count over window (assuming a processing-time attribute "proctime")
.window(Over.partitionBy("a").orderBy("proctime").preceding("unbounded_row").as("w"));
```
### 有界的 Over window
```java
// Bounded Event-time over window (assuming an event-time attribute "rowtime")
.window(Over.partitionBy("a").orderBy("rowtime").preceding("1.minutes").as("w"))

// Bounded Processing-time over window (assuming a processing-time attribute "proctime")
.window(Over.partitionBy("a").orderBy("proctime").preceding("1.minutes").as("w"))

// Bounded Event-time Row-count over window (assuming an event-time attribute "rowtime")
.window(Over.partitionBy("a").orderBy("rowtime").preceding("10.rows").as("w"))
 
// Bounded Processing-time Row-count over window (assuming a processing-time attribute "proctime")
.window(Over.partitionBy("a").orderBy("proctime").preceding("10.rows").as("w"))
```


# Data Types
Table API 构建在Flink’s DataSet and DataStream APIs 之上。 内部实现上依然是使用 TypeInformation 来定义数据类型。 所有支持类型定义在 `org.apache.flink.table.api.Types`。  下表简要列举了  Table API 类型， SQL类型 与 结果Java类之间的关系：


| Table API | SQL | Java type |
| - | - | - |
| Types.STRING |  VARCHAR | java.lang.String |
| Types.BOOLEAN | BOOLEAN | java.lang.Boolean |
| Types.BYTE |  TINYINT | java.lang.Byte |
| Types.SHORT | SMALLINT |  java.lang.Short |
| Types.INT | INTEGER, INT |  java.lang.Integer |
| Types.LONG |  BIGINT |  java.lang.Long |
| Types.FLOAT | REAL, FLOAT | java.lang.Float |
| Types.DOUBLE |  DOUBLE |  java.lang.Double |
| Types.DECIMAL | DECIMAL | java.math.BigDecimal |
| Types.SQL_DATE |  DATE |  java.sql.Date |
| Types.SQL_TIME |  TIME |  java.sql.Time |
| Types.SQL_TIMESTAMP | TIMESTAMP(3) |  java.sql.Timestamp |
| Types.INTERVAL_MONTHS | INTERVAL YEAR TO MONTH |  java.lang.Integer |
| Types.INTERVAL_MILLIS | INTERVAL DAY TO SECOND(3) | java.lang.Long |
| Types.PRIMITIVE_ARRAY | ARRAY | e.g. int[] |
| Types.OBJECT_ARRAY |  ARRAY | e.g. java.lang.Byte[] |
| Types.MAP | MAP | java.util.HashMap |
| Types.MULTISET |  MULTISET |  e.g. java.util.HashMap<String, Integer> for a multiset of String |
| Types.ROW | ROW | org.apache.flink.types.Row |

---

泛型和(嵌套)复合类型（POJOs, tuples, rows, Scala case classes） 也可以作为行的字段。任意嵌套层级的复合类型的字段可以通过访问函数的访问。
泛型可以作为一个黑盒，被传递给用户函数进行处理。


# 表达式语法

Some of the operators in previous sections expect one or more expressions. Expressions can be specified using an embedded Scala DSL or as Strings. Please refer to the examples above to learn how expressions can be specified.

前面提到的一些算子需要传一个或多个表达式。 表达式可以通过Scala DSL或Strings 来指定。

下面是 EBNF 表达式的语法：

```
expressionList = expression , { "," , expression } ;

expression = timeIndicator | overConstant | alias ;

alias = logic | ( logic , "as" , fieldReference ) | ( logic , "as" , "(" , fieldReference , { "," , fieldReference } , ")" ) ;

logic = comparison , [ ( "&&" | "||" ) , comparison ] ;

comparison = term , [ ( "=" | "==" | "===" | "!=" | "!==" | ">" | ">=" | "<" | "<=" ) , term ] ;

term = product , [ ( "+" | "-" ) , product ] ;

product = unary , [ ( "*" | "/" | "%") , unary ] ;

unary = [ "!" | "-" | "+" ] , composite ;

composite = over | suffixed | nullLiteral | prefixed | atom ;

suffixed = interval | suffixAs | suffixCast | suffixIf | suffixDistinct | suffixFunctionCall ;

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

nullLiteral = "Null(" , dataType , ")" ;

timeIntervalUnit = "YEAR" | "YEAR_TO_MONTH" | "MONTH" | "QUARTER" | "WEEK" | "DAY" | "DAY_TO_HOUR" | "DAY_TO_MINUTE" | "DAY_TO_SECOND" | "HOUR" | "HOUR_TO_MINUTE" | "HOUR_TO_SECOND" | "MINUTE" | "MINUTE_TO_SECOND" | "SECOND" ;

timePointUnit = "YEAR" | "MONTH" | "DAY" | "HOUR" | "MINUTE" | "SECOND" | "QUARTER" | "WEEK" | "MILLISECOND" | "MICROSECOND" ;

over = composite , "over" , fieldReference ;

overConstant = "current_row" | "current_range" | "unbounded_row" | "unbounded_row" ;

timeIndicator = fieldReference , "." , ( "proctime" | "rowtime" ) ;
```

Here, literal is a valid Java literal. String literals can be specified using single or double quotes. Duplicate the quote for escaping (e.g. 'It''s me.' or "I ""like"" dogs.").

The fieldReference specifies a column in the data (or all columns if * is used), and functionIdentifier specifies a supported scalar function. The column names and function names follow Java identifier syntax.

Expressions specified as strings can also use prefix notation instead of suffix notation to call operators and functions.

If working with exact numeric values or large decimals is required, the Table API also supports Java’s BigDecimal type. In the Scala Table API decimals can be defined by BigDecimal("123456") and in Java by appending a “p” for precise e.g. 123456p.

In order to work with temporal values the Table API supports Java SQL’s Date, Time, and Timestamp types. In the Scala Table API literals can be defined by using java.sql.Date.valueOf("2016-06-27"), java.sql.Time.valueOf("10:10:42"), or java.sql.Timestamp.valueOf("2016-06-27 10:10:42.123"). The Java and Scala Table API also support calling "2016-06-27".toDate(), "10:10:42".toTime(), and "2016-06-27 10:10:42.123".toTimestamp() for converting Strings into temporal types. Note: Since Java’s temporal SQL types are time zone dependent, please make sure that the Flink Client and all TaskManagers use the same time zone.

Temporal intervals can be represented as number of months (Types.INTERVAL_MONTHS) or number of milliseconds (Types.INTERVAL_MILLIS). Intervals of same type can be added or subtracted (e.g. 1.hour + 10.minutes). Intervals of milliseconds can be added to time points (e.g. "2016-08-10".toDate + 5.days).