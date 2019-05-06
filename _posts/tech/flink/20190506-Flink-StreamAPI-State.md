20190506-Flink-StreamAPI-State.md
参考
- [State & Fault Tolerance Overview]()
- [Working with State]()
- [The Broadcast State Pattern]()
- [Checkpointing]()
- [Queryable State]()
- [State Backends]()
- [State Schema Evolution]()
- [Custom State Serialization]()



# 概览
有状态的函数和算子可以跨事件保存数据， 使得状态成为复杂类型算子的一个重要的构建模块。
- 如果要搜索一个特定的事件模式，状态可以保存出现过的事件序列。
- 当需要聚合每分钟/小时/天的事件时， 状态可以保存实时合并的聚合结果。
- 当基于数据流训练机器学习模型时，状态可以保存当前版本的模型参数。
- 如果需要管理历史数据，状态可以高效地访问出现过的事件。

Flink 需要识别出状态，以便使用检查点来做状态的容错， 生成流应用的保存点。 

状态可以实现Flink应用的扩容， Flink 负责状态在多个并行实例间的再分发。

可查询状态允许在运行时从Flink外部访问状态。

使用状态时， 最好读一下[Flink状态后端的文档](https://ci.apache.org/projects/flink/flink-docs-release-1.8/ops/state/state_backends.html)。 Flink提供了不同的状态后端， 保存状态的方式与位置各有不同。 状态可以放在Java的堆内存或其它地方。 根据状态后端的不同，Flink可以管理很大体量的状态（可存放在内存，磁盘）。可以在不改变应用业务逻辑的前提下配置状态后端。

# 状态的使用
有两种基本的状态分类： 键状态（Keyed State）、 算子状态（Operator State）

## 键状态与算子状态
键状态总是关联到键的， 只能在 KeyedStream 的函数与算子中使用。

可以认为键状态是根据键进行分片了的算子状态。 每一个键状态逻辑上对应一个唯一的组合键值<parallel-operator-instance,key>， 因为每个键只属于键算子的一个并行实例，我们可以简单地认为是 <operator,key>。

键状态可以进一步组织为键组。 键组是Flink可以再分布键状态时的最小原子单位； 键组的数量和定义的最大并行度一致。 执行时， 键算子的每一个并行实例处理一个或多个键组。

算子状态（非键状态）， 每个算子状态对应一个并行算子的实例。 以 Kafka Connector  为例， Kafka consumer 的每一个并行实例都维护了一个保存 topic 的分片与 offset 的map作为算子状态。

当修改并行度时， 算子状态可以在并行的算子实例间再分配。 再分配时状态的模式也可以不一样。 

## 未加工的状态与被管理的状态（Raw and Managed State）
键状态与算子状态存在两种形式： 被管理的与未加工的。

被管理的状态是指定Flink运行时管理的数据结构表示的状态， 比如内部的 hash 表或 RocksDB。 比如 “ValueState”, “ListState”等。 Flink 运行时会编码这些状态并写入查检点。

未加工的状态是指算子以他们自己的数据结构来保存的状态。 生成检查点时，作为一个字节序列写入到检查点，Flink对它的数据结构一无所知只是看到一堆原始的字节。

所有的数据流函数都可以使用被管理的状态，但未加工状态的接口只能在实现算子时使用。 建议使用被管理的状态，因为调整并行度时Flink可以自动进行再分配，并且内存使用效率也更好。

如果想要自定义管理状态的序列化实现，请参考[自定义序列化指南](https://ci.apache.org/projects/flink/flink-docs-release-1.8/dev/stream/state/custom_serialization.html)

## 使用被管理的键状态
The managed keyed state interface provides access to different types of state that are all scoped to the key of the current input element. This means that this type of state can only be used on a KeyedStream, which can be created via stream.keyBy(…).

被管理的键状态接口提供了当前访问当前输入元素所属的键范围的不同类型的状态的方法。 也就是说这种类型的状态只能在由 stream.keyBy(…) 创建的 KeyedStream 上使用。

下面我们先看一下各种类型的状态。 然后再看看如何在程序中使用它们。 可用的状态有： 

- ValueState<T>: 保存一个值，可以更新和获取（使用范围限定为输入项对应的键， 所以算子内每个key都对应一个值）。 可以使用 update(T) 来更新， 使用 T value() 来获取。

- ListState<T>: 只在一个元素列表。 可以追加元素，或者获取一个当前保存的所有元素的 Iterable。 使用add(T) 或 addAll(List<T>)来增加元素， 使用 Iterable<T> get() 来获取 Iterable. 也可以使用 pdate(List<T>) 来覆盖当前的列表。

- ReducingState<T>: 保存所有添加到本状态上的元素的一个累计值。 接口类似 ListState， 通过add(T) 来添加元素，通过一个指定的ReduceFunction来将元素归约为一个聚合值。

- AggregatingState<IN, OUT>: 保存所有添加到本状态上的元素的一个累计值。 与ReducingState不同的是AggregatingState的聚合值的类型可以与添加到状态的输入元素的类型不一样。 接口类似ListState， 通过add(T) 来添加元素，通过一个指定的AggregateFunction来将元素归约为一个聚合值。

- FoldingState<T, ACC>: 与AggregatingState 类似，已经被标注为弃用。

- MapState<UK, UV>: 保存了一个映射列表。可以将键值对放入状态中， 也可获取一个当前全部映射的Iterable。 使用 put(UK, UV) 、 putAll(Map<UK, UV>) 来添加映射。  相应地可以使用entries(), keys() 和 values() 等方法来获取键或值。


所有类型的状态都有一个clear() 方法用来清除当前键对应的状态。

需要牢记： 一、状态对象只是用来连接操作状态的。 状态可能存放在磁盘或其它任何地方。 二、 取到的状态取决于输入元素对应的键的。 所以在UDF中每个事件处理时取得的状态可能是不同的（键不同）。

获取状态句柄需要创建一个StateDescriptor： 包含了状态名（需要是全局唯一的）、 状态值的类型、可能还有UDF如 ReduceFunction。 根据要使用的状态的类型，可以创建的有 ValueStateDescriptor, ListStateDescriptor, ReducingStateDescriptor, FoldingStateDescriptor 和 MapStateDescriptor。

状态通过RuntimeContext来访问，所以只能在 rich functions 中使用。  [详细参考这里](https://ci.apache.org/projects/flink/flink-docs-release-1.8/dev/api_concepts.html#rich-functions)。 RichFunction 获取到的 RuntimeContext 有如下方法可以访问状态：

- ValueState<T> getState(ValueStateDescriptor<T>)
- ReducingState<T> getReducingState(ReducingStateDescriptor<T>)
- ListState<T> getListState(ListStateDescriptor<T>)
- AggregatingState<IN, OUT> getAggregatingState(AggregatingStateDescriptor<IN, ACC, OUT>)
- FoldingState<T, ACC> getFoldingState(FoldingStateDescriptor<T, ACC>)
- MapState<UK, UV> getMapState(MapStateDescriptor<UK, UV>)

下面是一个FlatMapFunction的完整的例子：
```java


public class CountWindowAverage extends RichFlatMapFunction<Tuple2<Long, Long>, Tuple2<Long, Long>> {

    /**
     * ValueState 的句柄。 第一个字段是 count，第二个字段是 sum 。
     */
    private transient ValueState<Tuple2<Long, Long>> sum;

    @Override
    public void flatMap(Tuple2<Long, Long> input, Collector<Tuple2<Long, Long>> out) throws Exception {

        // 访问状态值
        Tuple2<Long, Long> currentSum = sum.value();

        // 更新 count
        currentSum.f0 += 1;

        // 将输入值加到第二个字段上
        currentSum.f1 += input.f1;

        // 更新状态
        sum.update(currentSum);

        // count 到达2 时，发送平均值并清除状态。
        if (currentSum.f0 >= 2) {
            out.collect(new Tuple2<>(input.f0, currentSum.f1 / currentSum.f0));
            sum.clear();
        }
    }

    @Override
    public void open(Configuration config) {
        ValueStateDescriptor<Tuple2<Long, Long>> descriptor =
                new ValueStateDescriptor<>(
                        "average", // 状态名
                        TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {}), // 类型信息
                        Tuple2.of(0L, 0L)); // 状态的默认值
        sum = getRuntimeContext().getState(descriptor);
    }
}

// 在流处理程序中就可以像这样使用 （假设已经有了StreamExecutionEnvironment env）
env.fromElements(Tuple2.of(1L, 3L), Tuple2.of(1L, 5L), Tuple2.of(1L, 7L), Tuple2.of(1L, 4L), Tuple2.of(1L, 2L))
        .keyBy(0)
        .flatMap(new CountWindowAverage())
        .print();

// 输出结果会是 (1,4) 和 (1,5)

```

这个例子实现了一个简单的计数窗口。 我们以元组的第一个字段为键（例子中都是1）。 函数将count和sum保存在ValueState中。 一旦count 达到2， 就会发送平均值并清空状态以便从0重新开始。 如果元组的第一个字段有不同的值， 对于不同的输入键， 会保存不同的状态值。

### 状态的过期时间 (TTL)

所有类型的键状态都可以指定一个过期时间。 如果配置了过期时间且状态值过期了， 保存的值将会尽可能地被清除掉，下面会详细讨论。

所有的状态集合都支持单元素过期的。 也就是说的列表的元素和map的条目是独立过期的。 

为了使用TTL状态，必须先构建现一个StateTtlConfig配置对象。 所有的状态 descriptor 都可以通过传这个配置对象来开启TTL功能。

```java
import org.apache.flink.api.common.state.StateTtlConfig;
import org.apache.flink.api.common.state.ValueStateDescriptor;
import org.apache.flink.api.common.time.Time;

StateTtlConfig ttlConfig = StateTtlConfig
    .newBuilder(Time.seconds(1))
    .setUpdateType(StateTtlConfig.UpdateType.OnCreateAndWrite)
    .setStateVisibility(StateTtlConfig.StateVisibility.NeverReturnExpired)
    .build();
    
ValueStateDescriptor<String> stateDescriptor = new ValueStateDescriptor<>("text state", String.class);
stateDescriptor.enableTimeToLive(ttlConfig);

```


这个配置有几个选项需要考虑:

newBuilder 方法的第一个参数是强制要传的， 是过期时间的值。

更新类型配置了什么时候刷新状态过期时间（默认OnCreateAndWrite）：

- StateTtlConfig.UpdateType.OnCreateAndWrite - 只有在创建或写入时
- StateTtlConfig.UpdateType.OnReadAndWrite - 加上读取时


The state visibility configures whether the expired value is returned on read access if it is not cleaned up yet (by default NeverReturnExpired):
状态可见性配置了过期蛤还没有清除的值在读取时是否返回（默认 NeverReturnExpired）。

- StateTtlConfig.StateVisibility.NeverReturnExpired - 不返回
- StateTtlConfig.StateVisibility.ReturnExpiredIfNotCleanedUp - 返回

若是NeverReturnExpired， 过期的状态值就像是从来没有存在过一样， 即使还没有被清除。 这个选项对于过期后数据必须立即不可用的场景非常有用， 如隐私敏感数据。

另外一个 ReturnExpiredIfNotCleanedUp 允许返回过期但是还未清除的数据。

- 状态后端将最后一次的修改时间戳与用户值保存在一起， 也就是说启用这个功能会增加状态存储空间的消耗。 堆内存的状态和后端在内存中保存一个额外的Java对象， 包含一个到用户状态的引用和一个原始类型的long值。 RocksDB状态后端为每一个value，列表的元素和map的条目添加一个8 bytes的时间戳。
- 当前TTLs只支持处理时间
- 重加载状态时， 如果之前没有配置TTL后来在desriptor中启用了TTL或者反之，都会引起兼容性错误和StateMigrationException
- TTL配置不是检查点或保存点的一部分。
- 只有用户值序列化器支持null值的时候，带TTL的 map state 才支持null。 如果序列化器不支持null 值， 可以用NullableSerializer包装一下， 就是要额外费些bytes。

#### 过期状态的清除
默认地，过期的值只会在被显示地读取到时才会被移除， 例如，调用 ValueState.value()

这也就是说，如果过期的状态一直没有被读取， 就不会被移除， 会导致状态一直增加。 以后的发布版本可能会修改这个问题。 

全量快照时清除

另外， 可以在全量快照时开启清除选项， 可以减小size。  根据当前的实现， 本地的状态并不会被清除， 但不会将过期的状态包含到快照里， 这样的话从前一个快照恢复后，过期的状态就相当于移除了。 可以通过StateTtlConfig来配置：
```java


import org.apache.flink.api.common.state.StateTtlConfig;
import org.apache.flink.api.common.time.Time;

StateTtlConfig ttlConfig = StateTtlConfig
    .newBuilder(Time.seconds(1))
    .cleanupFullSnapshot()
    .build();

```

这个选项不适用于 RocksDB 的增量检查点。

对于一个存在的作业， 清除策略可以在StateTtlConfig中随时开启或关闭，如从保存点重启。


另外一个方法是增量地触发清除。触发器可以通过其它状态的访问或者事件事件处理来回调。 如果这个清除策略对某个状态开启， 存储后端会为这个状态所有的条目保持一个懒全局的迭代器。 每当增量清除被触发， 这个迭代器就会推进。 穿过的状态条目就会被检查是否过期，过期的就会被清除。 可以通过StateTtlConfig来开启：

```java
import org.apache.flink.api.common.state.StateTtlConfig;
 StateTtlConfig ttlConfig = StateTtlConfig
    .newBuilder(Time.seconds(1))
    .cleanupIncrementally()
    .build();
```


This strategy has two parameters. The first one is number of checked state entries per each cleanup triggering. If enabled, it is always triggered per each state access. The second parameter defines whether to trigger cleanup additionally per each record processing.

Notes:

    If no access happens to the state or no records are processed, expired state will persist.
    Time spent for the incremental cleanup increases record processing latency.
    At the moment incremental cleanup is implemented only for Heap state backend. Setting it for RocksDB will have no effect.
    If heap state backend is used with synchronous snapshotting, the global iterator keeps a copy of all keys while iterating because of its specific implementation which does not support concurrent modifications. Enabling of this feature will increase memory consumption then. Asynchronous snapshotting does not have this problem.
    For existing jobs, this cleanup strategy can be activated or deactivated anytime in StateTtlConfig, e.g. after restart from savepoint.


# 广播状态模式
# 检查点
# 可查询的状态
# 状态后端
# 演进更新状态模式
# 自定义状态的序列化