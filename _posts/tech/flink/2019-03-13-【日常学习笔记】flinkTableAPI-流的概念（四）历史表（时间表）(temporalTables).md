---
layout: post
title:  "【日常学习笔记】flinkTableAPI-流的概念（四）历史表（时间表）(temporalTables)"
date:   2019-03-13 22:30:00 +0800
tags:
        - 流处理
        - java
        - Flink
---

> Temporal Tables 可以翻译作历史表，也可以叫时间表， 这里统一翻译为历史表。

历史表是一个概念，代表一个不断变化的历史表在一个特定时间点上的（参数化）视图。

Flink 可以跟踪底层仅追加表上的变更， 并允许查询表在某一时间点上的内容。

# Motivation
假设有下面一张汇率历史表：

```sql
SELECT * FROM RatesHistory;

rowtime currency   rate
======= ======== ======
09:00   US Dollar   102
09:00   Euro        114
09:00   Yen           1
10:45   Euro        116
11:15   Euro        119
11:49   Pounds      108
```
汇率历史表表示一个不断变化的对日元(汇率1)汇率的仅追加表。如欧元对日元汇率在【09:00 ~ 10:45】为114， 【10:45 ~ 11:15】 为116。

可能通过下面SQL查询【10:58】这一时间点所有的汇率数据：

```
SELECT *
FROM RatesHistory AS r
WHERE r.rowtime = (
  SELECT MAX(rowtime)
  FROM RatesHistory AS r2
  WHERE r2.currency = r.currency
  AND r2.rowtime <= TIME '10:58');
```

子查询查出相对币种的小于等于指定时间点的最大更新时间。 外层查询列出等于这些最大更新时间的汇率记录。


下表为查询结果。 取【10:45】欧元汇率的更新而不是【11:15】的更新作为【10:58】时点的欧元汇率。

```
rowtime currency   rate
======= ======== ======
09:00   US Dollar   102
09:00   Yen           1
10:45   Euro        116
```

历史表的概念是为了简化加速这种查询，并减小Flink使用的状态。 历史表是仅追加表上的一个参数化视图， 把仅追加表的数据记录当作是表的changelog, 并提供表在特定时间点上数据版本。 将仅追加表解释为changelog需要指定一个主键和一个时间属性。 主键决定记录的覆盖，记录 时间属性决定特定时间哪个记录版本是有效的。

在上面上例子中， currency是主键， rowtime 是时间属性。

在Flink中， 历史表通过历史表函数来表示。

# Temporal Table Functions

为了访问历史表中的数据，必须传一个时间参数来确定要返回的表数据版本。Flink使用表函数的SQL语法来表达。

一旦定义了， 历史表函数接收一个时间参数，并返回一个行的集合。 集合中包含了，基于给定的时间参数，所有存在的主键对应的最新版本的数据。

假设基于 RatesHistory表定义了一个历史表函数 Rates(timeAttribute)， 我们可以这样使用：
```
SELECT * FROM Rates('10:15');

rowtime currency   rate
======= ======== ======
09:00   US Dollar   102
09:00   Euro        114
09:00   Yen           1

SELECT * FROM Rates('11:00');

rowtime currency   rate
======= ======== ======
09:00   US Dollar   102
10:45   Euro        116
09:00   Yen           1
```
每个对Rates(timeAttribute)的查询会根据 timeAttribute 返回Rates的状态。

note: 当前 Flink 还不支持直接在历史表函数中指定固定的时间。 也就是说历史表函数只能用在连接查询中。 上面的例子只是为了直观的演示 Rates(timeAttribute) 的调用与返回。

## Defining Temporal Table Function
The following code snippet illustrates how to create a temporal table function from an append-only table.

下面代码演示如何从仅追加表创建一个历史表函数。

```java
import org.apache.flink.table.functions.TemporalTableFunction;
(...)

// 获取Stream 与 Table 环境
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
StreamTableEnvironment tEnv = TableEnvironment.getTableEnvironment(env);

// 模拟汇率历史数据
List<Tuple2<String, Long>> ratesHistoryData = new ArrayList<>();
ratesHistoryData.add(Tuple2.of("US Dollar", 102L));
ratesHistoryData.add(Tuple2.of("Euro", 114L));
ratesHistoryData.add(Tuple2.of("Yen", 1L));
ratesHistoryData.add(Tuple2.of("Euro", 116L));
ratesHistoryData.add(Tuple2.of("Euro", 119L));

// 使用上面的数据创建注册一张表
DataStream<Tuple2<String, Long>> ratesHistoryStream = env.fromCollection(ratesHistoryData);
Table ratesHistory = tEnv.fromDataStream(ratesHistoryStream, "r_currency, r_rate, r_proctime.proctime");

tEnv.registerTable("RatesHistory", ratesHistory);

// 创建并注册一个历史表函数
// 将 "r_proctime" 定义为时间属性， "r_currency" 为主键
TemporalTableFunction rates = ratesHistory.createTemporalTableFunction("r_proctime", "r_currency"); // <==== (1)
tEnv.registerFunction("Rates", rates);                                                              // <==== (2)
```
行(1) 创建一个可以在 TableAPI 中使用的历史表函数rates。
行(2) 将函数rates 以 名称"Rates" 注册到Table环境，以允许在 SQL 中使用 Rates 函数。
