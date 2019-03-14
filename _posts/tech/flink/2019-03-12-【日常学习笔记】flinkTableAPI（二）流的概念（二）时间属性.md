---
layout: post
title:  "【日常学习笔记】flinkTableAPI（二）流的概念（二）时间属性"
date:   2019-03-12 22:30:00 +0800
tags:
        - 流处理
        - java
        - Flink
---



Flink 能基于不同的时间定义来处理流数据。 

- Processing time ： 执行对应的操作时，机器的系统时间。
- Event time ： 附加在流上每条数据的时间戳, 一般可以是事件的发生时间。
- Ingestion time : 事件数据进入到 Flink 的时间。

这里主要介绍在 Flink TableAPI 和 SQL 的基于时间的处理中， 如何定义时间属性的。

# 关于时间属性的介绍
基于时间的操作（如表API和SQL中的窗口）都需要有关时间的定义及其来源的信息。这样表才可以提供逻辑时间属性，用于在程序中指示时间和访问相应的时间戳。

时间属性可以是每个表模式的一部分。 可以在从 DataStream 创建表时定义，也可以在 TableSource 中预定义。 一旦时间属性被定义后，就可以作为一个一般的字段来引用，也可以在基于时间的操作中使用。

只要时间属性没有被修改，并且只是从查询的一部分转发到另一部分，它就仍然是一个有效的时间属性。时间属性的行为类似于常规时间戳，可以访问这些属性进行计算。如果在计算中使用时间属性，它将具体化并成为常规时间戳。常规时间戳不与Flink的时间水位系统协同工作，因此不能再用于基于时间的操作。

Table 程序中需要为 StreamEnvironment 中指定相应的 time characteristic：
```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

env.setStreamTimeCharacteristic(TimeCharacteristic.ProcessingTime); // 默认为ProcessingTime

// 或者:
// env.setStreamTimeCharacteristic(TimeCharacteristic.IngestionTime);
// env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
```


# Processing time

Processing time 允许 Table 程序基于本地机器时间来生成结果。 它是最简单的时间定义，但却不保证结果的正确性。 即不需要提取时间戳，也不需要生成水位。

有两种方式来定义processing时间属性。

## During DataStream-to-Table Conversion

processing 时间属性在定义模式时通过 【.proctime】属性来定义， 只能通过为表的物理模式额外增加一个逻辑字段的方式来定义。 因此只能定义在模式定义的最后。

```java
DataStream<Tuple2<String, String>> stream = ...;

// 定义一个额外的逻辑字段作为 processing 时间属性
Table table = tEnv.fromDataStream(stream, "Username, Data, UserActionTime.proctime");

WindowedTable windowedTable = table.window(Tumble.over("10.minutes").on("UserActionTime").as("userActionWindow"));
```

## Using TableSource

通过实现DefinedProcTimeAttribute接口的TableSource来定义processing时间属性。逻辑时间属性附加到由TableSource的返回类型定义的物理模式中。

```java
// 定义一个带 processing 时间属性的table source
public class UserActionSource implements StreamTableSource<Row>, DefinedProctimeAttribute {

    @Override
    public TypeInformation<Row> getReturnType() {
        String[] names = new String[] {"Username" , "Data"};
        TypeInformation[] types = new TypeInformation[] {Types.STRING(), Types.STRING()};
        return Types.ROW(names, types);
    }

    @Override
    public DataStream<Row> getDataStream(StreamExecutionEnvironment execEnv) {
        // create stream
        DataStream<Row> stream = ...;
        return stream;
    }

    @Override
    public String getProctimeAttribute() {
        // 附加一个名为 “UserActionTime”的字段作为表的第三个字段
        return "UserActionTime";
    }
}

// register table source
tEnv.registerTableSource("UserActions", new UserActionSource());

WindowedTable windowedTable = tEnv
    .scan("UserActions")
    .window(Tumble.over("10.minutes").on("UserActionTime").as("userActionWindow"));
```

# Event time
Event time 允许 table 程序基于包含在每条数据中的时间来产生结果。 在乱序事件和延迟事件的场景得到一致性的结果。 从持久化存储中读取记录时，也可以保证结果的重放。

此外，event time 允许在批和流环境中为表程序使用统一的语法。流环境中的时间属性可以是批处理环境中记录的常规字段。

为了在流中处理乱序事件和识别延迟事件， Flink 需要从事件数据中提取时间戳，并标识时间的进展（水位）。

事件时间属性可以在 流转表时定义，也可以使用 TableSource时定义。



## During DataStream-to-Table Conversion


在定义模式时， 事件时间属性通过 【.proctime】属性来定义。 被转换的DataStream 必须是已经分配了时间戳和水位的。


流转表时有两种方式来定义时间属性。根据指定 【.rowtime】 的字段名在 DataStream 中是否存在， 时间戳字段：
- 作为一个新字段追加到模式中
- 或者替换掉已经存在的字段

两种方式下， 事件时间戳字段都持有 DataStream 的事件时间戳的值。

```java
// Option 1:

// 提取时间戳并生成水位
DataStream<Tuple2<String, String>> stream = inputStream.assignTimestampsAndWatermarks(...);

// 声明一个附加的逻辑字段做为事件时间属性
Table table = tEnv.fromDataStream(stream, "Username, Data, UserActionTime.rowtime");


// Option 2:

// 提取第一个字段为时间戳， 并生成水位
DataStream<Tuple3<Long, String, String>> stream = inputStream.assignTimestampsAndWatermarks(...);

// 第一个字段已经提取为事件时间，不再需要了
// 替换第一个属性为逻辑时间属性
Table table = tEnv.fromDataStream(stream, "UserActionTime.rowtime, Username, Data");

// Usage:

WindowedTable windowedTable = table.window(Tumble.over("10.minutes").on("UserActionTime").as("userActionWindow"));
```

## Using TableSource

通过实现了DefinedRowtimeAttributes接口的TableSource来指定事件时间属性。 getRowtimeAttributeDescriptors() 方法返回一个 RowtimeAttributeDescriptor 的 list 来 描述最终的 时间属性名称， 提取时间属性值的时间提取器， 和关联到时间属性的水位策略。


请确保 getDataStream() 方法返回的数据流与定义的时间属性对齐。只有在定义了 StreamRecordTimestamp 时间戳提取器时，才会考虑数据流的时间戳（由 TimestampAssigner 分配的时间戳）。只有在定义了 PreserveWatermarks 水印策略时，才会保留数据流的水印。否则，只有TableSource的rowtime属性的值是有意义的。

```java
// define a table source with a rowtime attribute
public class UserActionSource implements StreamTableSource<Row>, DefinedRowtimeAttributes {

    @Override
    public TypeInformation<Row> getReturnType() {
        String[] names = new String[] {"Username", "Data", "UserActionTime"};
        TypeInformation[] types =
            new TypeInformation[] {Types.STRING(), Types.STRING(), Types.LONG()};
        return Types.ROW(names, types);
    }

    @Override
    public DataStream<Row> getDataStream(StreamExecutionEnvironment execEnv) {
        // create stream
        // ...
        // assign watermarks based on the "UserActionTime" attribute
        DataStream<Row> stream = inputStream.assignTimestampsAndWatermarks(...);
        return stream;
    }

    @Override
    public List<RowtimeAttributeDescriptor> getRowtimeAttributeDescriptors() {
        // Mark the "UserActionTime" attribute as event-time attribute.
        // We create one attribute descriptor of "UserActionTime".
        RowtimeAttributeDescriptor rowtimeAttrDescr = new RowtimeAttributeDescriptor(
            "UserActionTime",
            new ExistingField("UserActionTime"),
            new AscendingTimestamps());
        List<RowtimeAttributeDescriptor> listRowtimeAttrDescr = Collections.singletonList(rowtimeAttrDescr);
        return listRowtimeAttrDescr;
    }
}

// register the table source
tEnv.registerTableSource("UserActions", new UserActionSource());

WindowedTable windowedTable = tEnv
    .scan("UserActions")
    .window(Tumble.over("10.minutes").on("UserActionTime").as("userActionWindow"));
```
