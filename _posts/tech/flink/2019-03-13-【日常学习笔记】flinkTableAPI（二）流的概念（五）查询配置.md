---
layout: post
title:  "【日常学习笔记】flinkTableAPI（二）流的概念（五）查询配置"
date:   2019-03-13 20:30:00 +0800
tags:
        - 流处理
        - java
        - Flink
---
无论输入是有界批输入还是无界流输入， Table API 和SQL查询都具有相同的语义。在许多情况下，对流输入的连续查询的计算能够得到与脱机计算相同的准确结果。但在一般情况下，这是不太可能达到，因为连续查询必须限制它们所维护的状态的大小，以避免耗尽存储空间，并且能够长时间处理无界的流数据。因此，根据输入数据和查询本身的特性，连续查询可能只能提供近似的结果。

Flink’s Table API 和 SQL 接口提供一些参数来调节持续查询的精确性与资源消耗。 这些参数骑过 QueryConfig 对象来指定。 QueryConfig 可能从 TableEnvironment 获取，并在 Table转化时回传，即转换为 DataStream时， 或 通过 TableSink 发送结果时。

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// 从 TableEnvironment 获取 查询配置
StreamQueryConfig qConfig = tableEnv.queryConfig();
// 设置查询参数
qConfig.withIdleStateRetentionTime(Time.hours(12), Time.hours(24));

// 定义查询
Table result = ...

// 创建 TableSink
TableSink<Row> sink = ...

// 注册 TableSink
tableEnv.registerTableSink(
  "outputTable",               // table name
  new String[]{...},           // field names
  new TypeInformation[]{...},  // field types
  sink);                       // table sink

// 通过 TableSink 发送 Table 结果时回传 qConfig
result.insertInto("outputTable", qConfig);

// 转换为 DataStream时 回传 qConfig 参数
DataStream<Row> stream = tableEnv.toAppendStream(result, Row.class, qConfig);

```

# Idel State Retention Time
许多查询根据一个或多个key来聚合或连接记录。 当这种查询在流上执行时， 持续查询需要为每一个key聚合记录或分片维护结果。 如果流数据中key的取值域是不断演化发展的，即，活跃的key值随着时间而变化， 随着越来越多 distinct 的key 出现，持续查询会堆积越来越多的状态。 然而key常常在过了一段时间后变成非活跃的，且其对应的状态也基本不再使用。

例如下面查询计算每个会话的点击数。

```sql
SELECT sessionId, COUNT(*) FROM clicks GROUP BY sessionId;
```

作为一个分组key, 持续查询会为每个 sessionId 维护一个计数状态。 sessionId 取值随着时间推移是不断推进的， 会话结束后（即一段时间后）对应的sessionId 值也会变为非活跃的。但是持续查询不可能知道sessionId 对应的会话什么时候结束， 只能假设其在未来任何时间点还会出现。 其为任何一个出现过的sessionId维护一个计数。 因此随着越来越多的sessionId值的出现，状态的总大小持续增长。

闲置状态保留时间（Idle State Retention Time）参数定义了不再被更新的状态移除前的保留时长。 像前面的例子， 一旦在配置的时长内没有被更新，sessionId 的计数会立即被移除。

key 一旦被移除，持续查询会完全忘记其出现过。 如果具相同的key的数据再出现，这条数据又会被当成这个key的第一条数据来处理。 以上面的例子，意味着 对应的sessionId 计数会从0再开始。

有两个参数来配置闲置状态保留时间：
- 最小闲置状态保留时间： 移除前的最小保留时长
- 最大闲置状态保留时间： 移除前的最大保留时长

```java
StreamQueryConfig qConfig = ...

// 设置闲置状态保留时间: min = 12 hours, max = 24 hours
qConfig.withIdleStateRetentionTime(Time.hours(12), Time.hours(24));
```

清除状态需要额外的管理成本，mintime和maxtime的差值越大，成本越低。mintime和maxtime的差值至少为5分钟。
