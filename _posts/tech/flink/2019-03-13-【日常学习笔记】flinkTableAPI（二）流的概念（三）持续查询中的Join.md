---
layout: post
title:  "【日常学习笔记】flinkTableAPI（二）流的概念（三）持续查询中的Join"
date:   2019-03-13 16:30:00 +0800
tags:
        - 流处理
        - java
        - Flink
---


在批处理中，数据的连接比较容易理解。但在动态表中没那么容易理解，甚至有些费解。

# 常规连接
常规Join是应用最广泛的Join类型， 输入join的两边的任何数据插入或更新都是可见的，且可以影响整体的Join结果。 如， 左边新插入一条数据， 这条数据会与右边之前和将来的数据进行联接。

```sql
SELECT * FROM Orders
INNER JOIN Product
ON Orders.productId = Product.id
```
这些语义允许所有更新类型（insert, update, delete）的输入表。

这种操作有个重要的影响点：需要将join两边的输入数据永久地保存到Flink 的状态中。 随着输入表数据的增长， 资源的使用也会无限地增长。

# 时间窗口的联接
有时间窗口的连接由联接谓词定义，该谓词检查输入记录的时间属性是否在某些时间约束内，即时间窗口。

```sql
SELECT *
FROM
  Orders o,
  Shipments s
WHERE o.id = s.orderId AND
      o.ordertime BETWEEN s.shiptime - INTERVAL '4' HOUR AND s.shiptime
```

与常规联接操作相比，这种联接只支持有时间属性的 append-only 表。由于时间属性是准单调递增的，因此Flink可以在不影响结果正确性的情况下将旧值从状态中删除。

# 使用历史表的连接
下面演示一个 append-only 订单表与持续更新的汇率历史表的连接。

订单表是一个 仅追加的表， 表示某一时间以某币种支付特定金额的支付单。
如在10:15支付了2欧元。
```sql
SELECT * FROM Orders;

rowtime amount currency
======= ====== =========
10:15        2 Euro
10:30        1 US Dollar
10:32       50 Yen
10:52        3 Euro
11:04        5 US Dollar
```

汇率历史表表示一个不断变化的对日元(汇率1)汇率的仅追加表。如欧元对日元汇率在【09:00 ~ 10:45】为114， 【10:45 ~ 11:15】 为116。
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

基于这些我们可以计算出所有订单转换为日元后的总金额。

如，对于下面的订单我们使用rowtime时对应的汇率（114）来转换订单金额。
```
rowtime amount currency
======= ====== =========
10:15        2 Euro
```

不使用历史表的话，这个查询会这样写
```sql
SELECT
  SUM(o.amount * r.rate) AS amount
FROM Orders AS o,
  RatesHistory AS r
WHERE r.currency = o.currency
AND r.rowtime = (
  SELECT MAX(rowtime)
  FROM RatesHistory AS r2
  WHERE r2.currency = o.currency
  AND r2.rowtime <= o.rowtime);
```

通过在 汇率历史表上构建一个历史表， 我们可以这样写SQL查询：
```sql
SELECT
  o.amount * r.rate AS amount
FROM
  Orders AS o,
  LATERAL TABLE (Rates(o.rowtime)) AS r
WHERE r.currency = o.currency
```


连接主表(左侧)的每条记录都会根据时间属性与从表(右侧)对应版本的记录连接。为了从表(右侧)的数据更新，从表需要定义一个主键。

在我们的例子中， 订单表的每条数据都会根据 订单的 rowtime 与汇率历史表对应版本的数据进行连接。 币种字段被定义为汇率表的主键并被用作连接字段。 如果查询是基于 processing-time 的， 新追加的订单会一直与最新版本的汇率进行连接。

与常规连接不同， 这意味从表的新数据不会影响之前的连接结果。这样也可以减少Flink必须存放在状态里的数据量。

与时间窗口的连接相比， 历史表连接不用定义时间窗口。 主表的记录永远根据指定的时间属性与从表对应版本的记录连接。  从表中的记录最终会变为旧值。 随着时间推移，记录的不再被使用的旧版本（对于特定主键）就可以从状态中删除了。

Such behaviour makes a temporal table join a good candidate to express stream enrichment in relational terms
这种行为使得历史表连接成为用关系术语来表示流丰富性的不错的选择。



## 用法
定义一个历史表函数后， 我们就可以使用了。  历史表函数可以与一般的表函数一样使用。
下面的代码演示了如何解决我们上面的问题：
SQL
```sql
SELECT
  SUM(o_amount * r_rate) AS amount
FROM
  Orders,
  LATERAL TABLE (Rates(o_proctime))
WHERE
  r_currency = o_currency
```
Java
```java
Table result = orders
    .join(new Table(tEnv, "rates(o_proctime)"), "o_currency = r_currency")
    .select("(o_amount * r_rate).sum as amount");
```
注意点： 历史表连接还没有实现在查询配置中定义的状态保持。这意味着随着历史记录表的主键数量规模增长，计算查询结果所需的状态可能会无限增长。

## 基于处理时间的历史表
基于处理时间的时间属性，不能将过去的时间历史表函数的参数。 根据定义，永远是当前的时间戳。 调用基于处理时间的历史表函数返回的数据一直是底层表的最后一个版本。底层表的任何更新会立即覆盖当前的值。

从表中只有最后一个版本（对应主键）的记录是只留在状态中的， 从表的更新不影响之前的连接结果。

可以把基于处理时间的历史表连接想象为一个存储了所有从表数据的 HashMap<K,V>。 当从表中新的记录的key与之前的数据相同，则新值会覆盖旧值。 主表中的记录永远是与 HashMap 中最新的值做连接。

## 基于事件时间的历史表
基于事件时间的时间属性，可以将已经过去的时间传入历史表函数。  这允许在某一个共同的时间点上对两张表进行连接。

相比于基于处理时间的历史表连接， 基于事件时间的历史表不仅保存从表数据的最后一个版本到状态中，而是保存最后一个时间水位后所有的数据版本（根据时间来标识）。

例如， 根据历史表定义，追加到主表的一个带有【12:30:00】事件时间戳的输入记录会与从表的中【12:30:00】时间点对应的数据版本进行连接。 也就是说， 输入数据只与对应主键的小于等于【12:30:00】时间的最新版本的数据连接。

根据事件时间的定义， 水位可以让连接操作向前推移时间，并忽略从表中不再需要的版本，因为不会再有水位时间之前的输入。