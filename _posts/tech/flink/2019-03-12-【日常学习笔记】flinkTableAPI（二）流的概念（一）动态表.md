---
layout: post
title:  "【日常学习笔记】flinkTableAPI（二）流的概念（一）动态表"
date:   2019-03-12 20:30:00 +0800
tags:
        - 流处理
        - java
        - Flink
---


SQL 和关系代数在设计上并没有考虑流数据。 因而关系代数（SQL）与流处理存在着概念上的缺失。
这里讨论一下这些差异，并解释 Flink 如何在无界数据流上实现常规数据库在有界数据上相同的的语义。

# 数据流上的关系查询

| 关系/SQL | 流处理 |
| ------ | ------ | ------ |
| 关系是有界的元组集合 | 流是无限的的元组序列 |
| 批查询的对象是全体数据 | 流查询无法访问到全体数据，需要等待数据不断流入 |
| 批查询的最终结果是确定的 | 流查询的结果随着数据的流入不断更新，永不结束 |



尽管存在这些差异，但使用关系查询和SQL处理流并非不可能。高级关系数据库系统提供一个称为物化视图的特性。物化视图定义为SQL查询，就像常规虚拟视图一样。与虚拟视图不同，物化视图缓存查询结果，这样在访问视图时就不需要计算查询。缓存的一个常见挑战是防止缓存提供过期数据。当修改其定义查询的基表时，物化视图将过期。急切视图维护是一种技术，它可以在基本表更新后立即更新物化视图。


如果我们考虑以下因素，那么在流上的热切视图维护和SQL查询之间的联系就会变得明显：

- 数据库表是INSERT, UPDATE, 和 DELETE DML语句流（changelog 流）的结果
- 物化视图被定义为 SQL 查询。 为了更新视图，查询持续的处理视图基本关系的 changelog 流。
- 物化视图是流查询的结果

考虑到这些要点，我们将在下一节中介绍以下动态表的概念。

# 动态表与持续查询

动态表是 Flink TableAPI 和SQL 支持流数据处理的核心概念。 与表示批处理数据的静态表不同，动态表随时间变化。它们可以像静态批处理表一样被查询。查询动态表会生成一个持续查询。持续查询不会终止并生成动态表。查询不断更新其（动态）结果表，以反映其（动态）输入表上的更改。本质上，动态表上的持续查询与定义物化视图的查询非常相似。

需要注意的是，持续查询的结果在语义上总是等价于在输入表的快照上以批处理模式执行的相同查询的结果。


下图显示了流、动态表和连续查询之间的关系:

[stream-query-stream.png](https://ci.apache.org/projects/flink/flink-docs-release-1.7/fig/table-streaming/stream-query-stream.png)

- 流转换为动态表
- 应用在动态表上的持续查询生成一个新的动态表
- 动态结果表转换回为一个流


Note: 动态表是一个逻辑概念。 执行查询时不需要（完全）具体化动态表。

下面我们将用下面模式的点击事件流来解释动态表和持续查询的概念：
```
[
  user:  VARCHAR,   // 用户名
  cTime: TIMESTAMP, // 点击时间
  url:   VARCHAR    // 访问 url
]
```

# Defining a Table on a Stream
为了使用关系型查询来处理流，需要将流转换为表。 概念上， 流的每条数据都被理解为对结果表的 INSERT 修改。 实质上，我们通过  INSERT-only changelog 流来构建一张表。


下图显示了点击事件流(左侧)如何转换为表(右侧)的。随着点击事件流数据的插入，结果表也持续的增长。

[Append mode](https://ci.apache.org/projects/flink/flink-docs-release-1.7/fig/table-streaming/append-mode.png)

Note: 内部实现上，定义在流上的表并没有具体化（物化）。

## 持续查询

持续查询应用动态表上，并生成新的动态表作为结果。 不同于批查询， 持续查询永不终止，并根据输入表的更新来更新结果表。 在任何时间点， 持续查询的结果在语义上等价于在相同的查询在输入表的快照上以批模式执行的结果。

下面我们展示对定义在点击事件流上的点击表进行查询的两个示例。

每一个查询是简单的 GROUP-BY COUNT 聚合查询。 它根据 user 字段将点击表分组，并统计 访问 URL 的数量。 下图展示了在更新点击表时，如何随着时间的推移对查询进行计算的。

[Continuous Non-Windowed Query](https://ci.apache.org/projects/flink/flink-docs-release-1.7/fig/table-streaming/query-groupBy-cnt.png)

当查询开始时， 点击表是空的。 当第一条数据插入到点击表时，查询开始计算结果表。 第一条数据 [Mary, ./home] 插入后， 结果表只有一条数据 [Mary, 1]。 第二条数据插入 [Bob, ./cart] 点击表时， 查询更新结果表并插入新一行数据 [Bob, 1]。   第三条数据 [Mary, ./prod?id=1] 将结果表中已经存在的行  [Mary, 1] 更新为  [Mary, 2]。 第四条数据被追加到点击表时， 查询插入第三行数据 [Liz, 1] 到结果表中。

第二个查询类似于第一个， 但除以 user 属性分组外，在计算 url 数目前，还加入了一个 一小时的滚动窗口（基于时间的计算如窗口，需要指定时间属性， 后面会讨论）。 下图展示了在不同时间点的输入输出来显示动态表的性质：


[Continuous Group-Window Query](https://ci.apache.org/projects/flink/flink-docs-release-1.7/fig/table-streaming/query-groupBy-window-cnt.png)

每个小时，查询持续计算结果并更新结果表。时间(cTime)介于12:00:00~12:59:59时，点击表包含四条数据。查询根据输入计算出两条结果行（每个用户一个），并将它们追加到结果表中。在下一个窗口【13:00:00~13:59:59】，点击表包含三条数据，导致另外两行添加到结果表中。随着时间的推移，更多的行被追加到点击表中，结果表也被更新。


## Update and Append Queries

上面两个例子似乎很相似，但有一个重要的不同点：
- 第一个查询会更新前面发送的结果，即， 定义结果表的 changelog 流 包含了 INSERT 和 UPDATE 变更。
- 第二个查询对结果表只有追加， 即， 结果表的 changelog 流只有 INSERT 类型的变更。


查询产生的是 append-only 表还是 updated 表的影响：
- 产生 updata 变更的查询通常需要维护更多的状态
- append-only 表到流的转换与 updated 表的转换不一样。


## 查询约束

许多（但不是全部）语义上有效的查询可以作为连续查询应用在流上进行计算。有些查询的计算成本很高，要么是因为它们需要维护的状态太大，要么是因为更新计算太昂贵。

- 状态大小： 连续查询是在无边界的流上进行的，通常会运行数周或数月。因此，连续查询处理的数据总量可能非常大。为了更新以前发出的结果，查询需要维护所有发出的行。例如，第一个示例查询需要存储每个用户的URL计数，以便在输入表接收到新行时增加计数并发送新结果。如果只跟踪注册用户，则要维护的计数数可能不会太高。但是，如果非注册用户分配了唯一的用户名，则要维护的计数数将随着时间的推移而增加，并可能最终导致查询失败。
```sql
SELECT user, COUNT(url)
FROM clicks
GROUP BY user;
```
- 更新计算： Some queries require to recompute and update a large fraction of the emitted result rows even if only a single input record is added or updated. Clearly, such queries are not well suited to be executed as continuous queries. An example is the following query which computes for each user a RANK based on the time of the last click. As soon as the clicks table receives a new row, the lastAction of the user is updated and a new rank must be computed. However since two rows cannot have the same rank, all lower ranked rows need to be updated as well.
```sql
SELECT user, RANK() OVER (ORDER BY lastAction)
FROM (
  SELECT user, MAX(cTime) AS lastAction FROM clicks GROUP BY user
);
```
[查询配置](https://ci.apache.org/projects/flink/flink-docs-release-1.7/dev/table/streaming/query_configuration.html)讨论了控制连续查询执行的参数。一些参数可以用来调整保持状态的大小与结果准确性。

# 表到流的转换 

与常规的数据库表一样， 动态表也可以持续的被 INSERT， UPDATE 和 DELETE 类型的变更更新。 可以是一个只有一条数据的表，不断地被更新， 也可以是一个没有 UPDATE和DELETE的 insert-only 表。

把动态表转换为流或者写到外部系统时， 这些变更需要被编码。 Flink 的 TableAPI与SQL支持三种编码方式：

- Append-only 流: 只被 INSERT 类型的变更修改的动态表可以转换为发送插入的数据行的流。

- Retract 流: retract 流包含两类消息，add 消息与 retract 消息。 动态表转换为 retract 流： INSERT 型变更编码为 add 消息； DELETE 型变更编码为 retract 消息； UPDATE 型变更编码为 一个 retract 消息（被更新的行） + 一个 add 消息（新数据的行）。 如下图所示：
[Dynamic tables](https://ci.apache.org/projects/flink/flink-docs-release-1.7/fig/table-streaming/undo-redo-mode.png)

- Upsert 流: Upsert 流包含两类消息， upsert 消息与 delete 消息。 动态表转换为 upsert 流需要一个唯一键(可以是组合键)： INSERT 与 UPDATE 型变更编码为 upsert 消息； DELETE 型变更编码为 delete 消息。 UPDATE 型变更编码为 一个 retract 消息（被更新的行） + 一个 add 消息（新数据的行）。 流的消费算子需要识别唯一键从而正确地处理消息。 与 retract 流主要的不同在于， UPDATE 被编码为一个消息， 因而更高效。如下图所示：

[Dynamic tables](https://ci.apache.org/projects/flink/flink-docs-release-1.7/fig/table-streaming/redo-mode.png)

动态表转流的API参考 [基本概念]()。 注意： 

注意，动态表转换为数据流时，只支持 append-only 流 和  retract 流。向外部系统发送动态表数据的接口TableSink 参考[这里]()。
