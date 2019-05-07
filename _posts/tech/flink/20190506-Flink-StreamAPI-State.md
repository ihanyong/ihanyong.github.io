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

## 原生的状态与托管的状态（Raw and Managed State）
键状态与算子状态存在两种形式： 托管的与原生的。

托管的状态是指定Flink运行时管理的数据结构表示的状态， 比如内部的 hash 表或 RocksDB。 比如 “ValueState”, “ListState”等。 Flink 运行时会编码这些状态并写入查检点。

原生的状态是指算子以他们自己的数据结构来保存的状态。 生成检查点时，作为一个字节序列写入到检查点，Flink对它的数据结构一无所知只是看到一堆原始的字节。

所有的数据流函数都可以使用托管的状态，但原始状态的接口只能在实现算子时使用。 建议使用托管的状态，因为调整并行度时Flink可以自动进行再分配，并且内存使用效率也更好。

如果想要自定义管理状态的序列化实现，请参考[自定义序列化指南](https://ci.apache.org/projects/flink/flink-docs-release-1.8/dev/stream/state/custom_serialization.html)

## 使用托管的键状态
The managed keyed state interface provides access to different types of state that are all scoped to the key of the current input element. This means that this type of state can only be used on a KeyedStream, which can be created via stream.keyBy(…).

托管的键状态接口提供了当前访问当前输入元素所属的键范围的不同类型的状态的方法。 也就是说这种类型的状态只能在由 stream.keyBy(…) 创建的 KeyedStream 上使用。

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

##### 全量快照时清除

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


##### 增量清除
另外一个方法是增量地触发清除。触发器可以通过其它状态的访问或者事件事件处理来回调。 如果这个清除策略对某个状态开启， 存储后端会为这个状态所有的条目保持一个懒全局的迭代器。 每当增量清除被触发， 这个迭代器就会推进。 穿过的状态条目就会被检查是否过期，过期的就会被清除。 可以通过StateTtlConfig来开启：

```java
import org.apache.flink.api.common.state.StateTtlConfig;
 StateTtlConfig ttlConfig = StateTtlConfig
    .newBuilder(Time.seconds(1))
    .cleanupIncrementally(cleanupSize, runCleanupForEveryRecord) //int, boolean
    .build();
```

这个策略有两个参数， 第一个是数量每次触发清除时需要检查的状态数量。如果开启的话每次访问状态都会触发。第二个定义是否每次处理数据时都触发增量清除。

注意：

- 如果没有状态访问发生或者也没有流数据被处理， 过期的状态会一直存在
- 增量清除会导致流处理延迟
- 当前只有基于堆内存的状态后端实现了增量清除。 对RocksDB进行配置无效。
- 如果使用同步快照的方式， 全局的迭代器会保存一份所有的键的拷贝， 因为当前的实现不支持并发修改。 这样会增加内存的消耗。 异步内存不存在这个问题。
- 对于已存在的作业， 清除策略任何时候都可以在StateTtlConfig中开启或关闭， 如从保存点重启


##### RocksDB 压缩时清除

RocksDB 状态后端还有另外一个清除策略， 通过激活Flink压缩过滤器来实现。 RocksDB 周期性地进行异步压缩来合并状态更新减小存储空间的占用。 Flink 压缩过滤器会检查TTL状态实体的时间戳并排除过期的值。

默认这个功能是关闭的。 必须通过FLink的配置  **state.backend.rocksdb.ttl.compaction.filter.enabled** 或者在作业中自定义RocksDB状态后端时调用 **RocksDBStateBackend::enableTtlCompactionFilter** 来开启 。 所有的TTL状态都可以配置使用这个过滤器：

```java
import org.apache.flink.api.common.state.StateTtlConfig;

StateTtlConfig ttlConfig = StateTtlConfig
    .newBuilder(Time.seconds(1))
    .cleanupInRocksdbCompactFilter()
    .build();
```

每当处理一定数量的状态实体后（默认1000）压缩过滤器会查询当前的时间戳检查是否过期。  可以通过 **StateTtlConfig.newBuilder(...).cleanupInRocksdbCompactFilter(long queryTimeAfterNumEntries)** 修改这个数量。  更频繁地执行可以提升的状态清除的速度，但也会拖慢压缩的性能。

可以打开FlinkCompactionFilter的 debug log : 

```
log4j.logger.org.rocksdb.FlinkCompactionFilter=DEBUG
```

注意： 
- 压缩时调用TTL过滤器会降低处理速度。 TTL过滤器要为正在压缩的每一个key的每一个状态条目转换时间戳检查是否过期。 对于集合类型的状态（列表与Map） 对于每个集合元素都要进行检查。
- 如果list状态的元素没有固定大小， TTL过滤器需要额外地为每一个状态条目（至少是从第一个过期元素到第一个未过期元素）通过JNI来调用一次Flink的java类型序列化器。
- 对于已存在的作业， 清除策略任何时候都可以在StateTtlConfig中开启或关闭， 如从保存点重启

### State in the Scala DataStream API
函数提供了一个便利方法来获取一个 valueState （Option），且必须返回一个更新值。 

```scala
val stream: DataStream[(String, Int)] = ...

val counts: DataStream[(String, Int)] = stream
  .keyBy(_._1)
  .mapWithState((in: (String, Int), count: Option[Int]) =>
    count match {
      case Some(c) => ( (in._1, c), Some(c + in._2) )
      case None => ( (in._1, 0), Some(in._2) )
    })
```

## 托管的算子状态的使用
To use managed operator state, a stateful function can implement either the more general CheckpointedFunction interface, or the ListCheckpointed<T extends Serializable> interface.

要使用托管的算子状态， 有状态的用户函数可以实现更泛化的CheckpointedFunction接口或 ListCheckpointed<T extends Serializable>接口。


#### CheckpointedFunction
CheckpointedFunction 接口提供了访问非键状态的方案。 定义了两个方法：

```java
void snapshotState(FunctionSnapshotContext context) throws Exception;

void initializeState(FunctionInitializationContext context) throws Exception;
```

需要执行检查点处理时，snapshotState() 会被调用。  每当UDF初始化时（函数第一次初始化或从检查点重加载时）会调用initializeState()。 也就是说，nitializeState()  不仅是各种类型的状态初始化的地方，也是包括状态的恢复逻辑。

当前，支持list-style的托管算子状态。状态应该是可序列化对象的列表，彼此独立，因此在重新扩容时可以重新分配。换句话说，这些对象是可以重新分布非键状态的最佳粒度。根据状态访问方法，定义了以下重新分发方案：

- 均匀分割再分配： 每个算子会返回一个状态元素的列表。 完整的状态在逻辑上就是所有的列表连接在一起。 在重新加载或重新分发时， 列表会被均匀地分割成若干子列表（算子并行度的数目）。 每个算子的实例都会得到一个子列表。 子列表可以为空，可能包含一个或多个元素。 例如， 并行度为1的算子查检点状态有两个元素A和B， 当并行度调整为2时，可能会元素A分配到算子实例0， 元素B分配到算子实例1。
-  联合再分配： 每个算子会返回一个状态元素的列表。 完整的状态在逻辑上就是所有的列表连接在一起。 在重新加载或重新分发时， 每个算子会得到一个完整的状态列表。

下面的例子是一个有状态的SinkFunction， 在发送数据出去前使用CheckpointedFunction 来缓存数据。  这里演示的是一个基本的均匀分割再分发：

```java


public class BufferingSink
        implements SinkFunction<Tuple2<String, Integer>>,
                   CheckpointedFunction {

    private final int threshold;

    private transient ListState<Tuple2<String, Integer>> checkpointedState;

    private List<Tuple2<String, Integer>> bufferedElements;

    public BufferingSink(int threshold) {
        this.threshold = threshold;
        this.bufferedElements = new ArrayList<>();
    }

    @Override
    public void invoke(Tuple2<String, Integer> value, Context contex) throws Exception {
        bufferedElements.add(value);
        if (bufferedElements.size() == threshold) {
            for (Tuple2<String, Integer> element: bufferedElements) {
                // send it to the sink
            }
            bufferedElements.clear();
        }
    }

    @Override
    public void snapshotState(FunctionSnapshotContext context) throws Exception {
        checkpointedState.clear();
        for (Tuple2<String, Integer> element : bufferedElements) {
            checkpointedState.add(element);
        }
    }

    @Override
    public void initializeState(FunctionInitializationContext context) throws Exception {
        ListStateDescriptor<Tuple2<String, Integer>> descriptor =
            new ListStateDescriptor<>(
                "buffered-elements",
                TypeInformation.of(new TypeHint<Tuple2<String, Integer>>() {}));

        checkpointedState = context.getOperatorStateStore().getListState(descriptor);

        if (context.isRestored()) {
            for (Tuple2<String, Integer> element : checkpointedState.get()) {
                bufferedElements.add(element);
            }
        }
    }
}


```

initializeState 方法接收一个 FunctionInitializationContext 参数，用于初始化 非键状态的容器。 这里是一个ListState类型的容器用来在生成检查点时保存非键状态对象。

非键状态的初始化与键状态类似，用一个带有状态名和值类型信息的 StateDescriptor来定义状态。

```java


ListStateDescriptor<Tuple2<String, Integer>> descriptor =
    new ListStateDescriptor<>(
        "buffered-elements",
        TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {}));

checkpointedState = context.getOperatorStateStore().getListState(descriptor);


```

状态访问方法的命名习惯是包含再分发模式与状态结构类型的。 例如， 使用getUnionListState(descriptor) 访问联合再分发的列表状态。 如果方法名中没有再分发模式，一般就是使用均匀再分发的意思，如 getListState(descriptor)。 

另外，也可以在initializeState()中初始化 键状态。 使用FunctionInitializationContext来完成。


#### ListCheckpointed
ListCheckpointed　是CheckpointedFunction　带有限制的变种，只支持在重加载时进行even-split 重分配的 list-style状态。　也是需要实现两个方法：

```java

List<T> snapshotState(long checkpointId, long timestamp) throws Exception;

void restoreState(List<T> state) throws Exception;
```
snapshotState()  需要返回一个对象list给查检点， 的恢复时restoreState接收到List. 如果状态不是可再分片的， 可以在  snapshotState() 中返回Collections.singletonList(MY_STATE) 。

### 有状态的源函数

与其它算子相比， 有状态的源算子需要更小心地处理。  为了保证状态更新与输出集合的原子性（失败/恢复时的一致性语义需求），用户需要从源上下文获取锁。

与其他运算符相比，有状态源需要更加小心。为了对状态和输出集合进行原子更新（在失败/恢复时只需要一次语义），用户需要从源上下文获取锁。

```java


public static class CounterSource
        extends RichParallelSourceFunction<Long>
        implements ListCheckpointed<Long> {

    /**  current offset for exactly once semantics */
    private Long offset = 0L;

    /** flag for job cancellation */
    private volatile boolean isRunning = true;

    @Override
    public void run(SourceContext<Long> ctx) {
        final Object lock = ctx.getCheckpointLock();

        while (isRunning) {
            // output and state update are atomic
            synchronized (lock) {
                ctx.collect(offset);
                offset += 1;
            }
        }
    }

    @Override
    public void cancel() {
        isRunning = false;
    }

    @Override
    public List<Long> snapshotState(long checkpointId, long checkpointTimestamp) {
        return Collections.singletonList(offset);
    }

    @Override
    public void restoreState(List<Long> state) {
        for (Long s : state)
            offset = s;
    }
}
```

一些卡算子需要查检点全部完成时的信息，可以参考下 **org.apache.flink.runtime.state.CheckpointListener** 接口


# 广播状态模式

还有一种广播状态，  可以用来满足这类用例： 一个流中的输入数据需要广播给所有的下游任务，然后存储在本地并用于处理所有其它的流的输入数据。 比如 一个吞吐量不高的包含需要应用到其它流数据的业务规则的流。 广播状态与其它的算子状态有如下不同：
1. 是Map格式
2. 只有在拥有一个广播流和一个非广播流作为输入流的算子可用
3. 算子可以有多个不同名称的广播状态

## 提供的 APIs
先看这么一个例子， 有一个各种颜色和形状的图形的流， 需要从中以特定的模式找出相同颜色的一对图形， 如三角形跟着一个矩形。 假设这个模式在运行时是需要变化的。

在这个例子中， 第一个是有颜色和开关两个属性的对象的流Items。 另外一个流是模式的流Rules。
对于Items 流， 以颜色为键进行分片，这样就可以确保相同颜色的图形会发送到同一台物理机器上。




```java
// key the shapes by color
KeyedStream<Item, Color> colorPartitionedStream = shapeStream
                        .keyBy(new KeySelector<Shape, Color>(){...});
```

对于Rules流数据，需要广播到所有的下游任务，且任务需要将数据保存在本地并应用到所有的输入数据上。 下面的片段 1. 广播rules流 并 2. 使用提供的 MapStateDescriptor创建存放规则的广播状态。


```java
// a map descriptor to store the name of the rule (string) and the rule itself.
MapStateDescriptor<String, Rule> ruleStateDescriptor = new MapStateDescriptor<>(
            "RulesBroadcastState",
            BasicTypeInfo.STRING_TYPE_INFO,
            TypeInformation.of(new TypeHint<Rule>() {}));
        
// broadcast the rules and create the broadcast state
BroadcastStream<Rule> ruleBroadcastStream = ruleStream
                        .broadcast(ruleStateDescriptor);

```

为了对Items流的输入数据应用规则，需要：

1. 连接两个流
2. 定义检测逻辑

可以在非广播流 (keyed 或 non-keyed)上调用 connect(BroadcastStream) 方法来连接一个广播流。 该方法返回一个BroadcastConnectedStream， 在上面可以调用 process() 来指定一个 CoProcessFunction。 这个UDF包含我们的匹配逻辑。 UDF具体的类型取决于非广播流的类型：
- 如果是有键的， UDF 是 KeyedBroadcastProcessFunction
- 如果是非键的， UDF 是 BroadcastProcessFunction

注意， 是在非广播流上以广播流作为参数调用connect()方法。
```java
DataStream<String> output = colorPartitionedStream
                 .connect(ruleBroadcastStream)
                 .process(
                     
                     // 类型参数分别是： 
                     //   1. 有键流的键的类型 the key of the keyed stream
                     //   2. 非广播流的元素类型
                     //   3. 广播流的元素类型
                     //   4.结果类型，这里是 string
                     
                     new KeyedBroadcastProcessFunction<Color, Item, Rule, String>() {
                         // 匹配逻辑
                     }
                 );

```

### BroadcastProcessFunction 和 KeyedBroadcastProcessFunction
与 CoProcessFunction 类似， 这两个function 也要实现两个方法； processBroadcastElement() 是用来处理广播流的输入数据的； processElement() 是用来处理非广播流数据的。 


```java
public abstract class BroadcastProcessFunction<IN1, IN2, OUT> extends BaseBroadcastProcessFunction {

    public abstract void processElement(IN1 value, ReadOnlyContext ctx, Collector<OUT> out) throws Exception;

    public abstract void processBroadcastElement(IN2 value, Context ctx, Collector<OUT> out) throws Exception;
}

public abstract class KeyedBroadcastProcessFunction<KS, IN1, IN2, OUT> {

    public abstract void processElement(IN1 value, ReadOnlyContext ctx, Collector<OUT> out) throws Exception;

    public abstract void processBroadcastElement(IN2 value, Context ctx, Collector<OUT> out) throws Exception;

    public void onTimer(long timestamp, OnTimerContext ctx, Collector<OUT> out) throws Exception;
}
```

两个UDF都要实现 processBroadcastElement() processElement() 来分别处理广播与非广播流的数据。 
两个方法的不同之处在于他们提供的上下文。 非广播的是ReadOnlyContext， 广播的是 Context。 

两个上下文都可以:

1. 都可以访问广播状态：ctx.getBroadcastState(MapStateDescriptor<K, V> stateDescriptor)
2. 都可以查询元素的时间戳： ctx.timestamp()
3. 获取当前的水位： ctx.currentWatermark()
4. 获取当前的处理时间： ctx.currentProcessingTime()
5. 向侧输出流发送数据元素: ctx.output(OutputTag<X> outputTag, X value)

ctx. getBroadcastState(stateDescriptor) 的stateDescriptor 要与上面 rules..broadcast(ruleStateDescriptor)中的ruleStateDescriptor 相同。

两个上下文不同于： 广播侧对状态是可读写访问，非广播侧是只读访问。  这是因为在Flink中没有跨任务的通信。为了保证所有的算子中看到的广播状态都是一样的， 去们只允许在广播侧读写广播状态、算子所有的任务看到的广播状态都是一样的，并且要求所有的任务对于每个传入的广播数据的计算也是一致的。如果不这样做的话， 就无法保证状态的一致性， 引起不一致的结果，调试起来也很困难。 

processBroadcast() 的实现逻辑在所有的并行实例上的行为应该是明确且一致的。

Finally, due to the fact that the KeyedBroadcastProcessFunction is operating on a keyed stream, it exposes some functionality which is not available to the BroadcastProcessFunction. That is:
因为KeyedBroadcastProcessFunction 是应用于有键的流的，与 BroadcastProcessFunction 相比提供了一些额外的功能：
1. processElement()中的ReadOnlyContext可以访问Flink底层的计时服务，允许注册事件/处理时间的计时器。 如果一个计时器被触发了， onTimer() 方法会被调用，并会接收到一个 OnTimerContext，这除了有ReadOnlyContext的功能外还能：
        + 判断触发计时器的是事件时间还是处理时间
        + 获取关联到计时器的键
2. processBroadcastElement() 的 Context 有一个 applyToKeyedState(StateDescriptor<S, VS> stateDescriptor, KeyedStateFunction<KS, S> function) 方法。 允许注册一个 KeyedStateFunction 应用到stateDescriptor关联的所有键的所有状态上。



注意:只能在 `KeyedBroadcastProcessFunction` 的 `processElement()` 方法中注册计时器。 `processBroadcastElement()` 不可以, 因为广播元素没有关联的Key。

回到上面的例子中， KeyedBroadcastProcessFunction 应该是这样的： 

```java
new KeyedBroadcastProcessFunction<Color, Item, Rule, String>() {

    // store partial matches, i.e. first elements of the pair waiting for their second element
    // we keep a list as we may have many first elements waiting
    private final MapStateDescriptor<String, List<Item>> mapStateDesc =
        new MapStateDescriptor<>(
            "items",
            BasicTypeInfo.STRING_TYPE_INFO,
            new ListTypeInfo<>(Item.class));

    // identical to our ruleStateDescriptor above
    private final MapStateDescriptor<String, Rule> ruleStateDescriptor = 
        new MapStateDescriptor<>(
            "RulesBroadcastState",
            BasicTypeInfo.STRING_TYPE_INFO,
            TypeInformation.of(new TypeHint<Rule>() {}));

    @Override
    public void processBroadcastElement(Rule value,
                                        Context ctx,
                                        Collector<String> out) throws Exception {
        ctx.getBroadcastState(ruleStateDescriptor).put(value.name, value);
    }

    @Override
    public void processElement(Item value,
                               ReadOnlyContext ctx,
                               Collector<String> out) throws Exception {

        final MapState<String, List<Item>> state = getRuntimeContext().getMapState(mapStateDesc);
        final Shape shape = value.getShape();
    
        for (Map.Entry<String, Rule> entry :
                ctx.getBroadcastState(ruleStateDescriptor).immutableEntries()) {
            final String ruleName = entry.getKey();
            final Rule rule = entry.getValue();
    
            List<Item> stored = state.get(ruleName);
            if (stored == null) {
                stored = new ArrayList<>();
            }
    
            if (shape == rule.second && !stored.isEmpty()) {
                for (Item i : stored) {
                    out.collect("MATCH: " + i + " - " + value);
                }
                stored.clear();
            }
    
            // there is no else{} to cover if rule.first == rule.second
            if (shape.equals(rule.first)) {
                stored.add(value);
            }
    
            if (stored.isEmpty()) {
                state.remove(ruleName);
            } else {
                state.put(ruleName, stored);
            }
        }
    }
}
```

## 重要注意事项

- 没有跨任务的通信： 如前所述，只有广播流的一侧的 (Keyed)-BroadcastProcessFunction 可以修改广播状态。 另外， 用户必须确保所有修改广播状态的任务节点上都要以相同的方式处理每一个输入事件， 否则会因为不同的任务节点上的状态内容不同而导致不一致的结果。

- 任务之间广播状态的事件顺序可能不一样： 尽管Flink会保证所有的广播事件会最终发送到所有的下游任务节点， 但事件到达的顺序可能是不一样的。 所以广播状态的更新一定不能依赖于输入事件的顺序。

- 所有的任务会将广播状态保存到检查点： 尽管所有Task上的广播状态是一样的， 生成检查点是每个任务是会独立地将广播状态写入自己的检查点中，而不是只要一个任务处理。 这样设计是为了避免所有的任务都从同一个文件恢复（会导致热点问题）， 尽管这样会有数据冗余占用了更多的存储空间。  Flink保证在恢复和调整容量时不会有数据重复或丢失。 当使用相同或更小的并行度恢复任务时， 每个任务都只需要读取自己的状态快照。 当扩容时， 已存在的任务读取自己的状态， 其它的任务（新的或故障转移的）以轮询的方式来读取检查点。 

- 没有 RocksDB 状态后端： 运行时广播状态保存在内存中，相应地从内存中分配空间

# 检查点
Flink 中的每个算子和函数可以拥有状态（详细参考[状态的使用]()）。 有状态的函数可以跨事件存储数据， 从而实现更复杂的功能。 

为了使状态具有容错能力， Flink为状态生成检查点。 检查点允许Flink能恢复状态和流处理的位置，从而使应用有了无错的执行语义。

详细参考 [状态与检查点容错]()



## 预备知识

Flink 的检查点机制为流和状态提供了持久化的存储， 一般需要：
- 一个持久化的数据源，保证一段时间内可以重放数据。 例如， 一些持久化的消息队列 （Kafka, RabbitMQ等）， 或者文件系统（HDFS， S3, GFS 等）
- 用来持久化状态的存储位置。 一般是分布式文件系统（HDFS， S3, GFS等）

## 启用配置检查点
检查点默认是关闭的。 可以在 StreamExecutionEnvironment 上调用 enableCheckpointing(n) 来开启检查点， n 是触发检查点的时间间隔（毫秒）。

检查点的其它配置参数:


- exactly-once vs. at-least-once: enableCheckpointing(n) 的可选参数。 Exactly-once 适合大多数的应用。 At-least-once 可能更适合那些追求低延迟的。

- checkpoint timeout: 如果生成检查点的处理超过这个时间，就会中止当前检查点的处理。


- minimum time between checkpoints: 上一次检查点结束到下一次检查点开始的时间间隔。 配置了这个参数也就意味着检查点触发间隔实际是大于这个值， 也意味着并发处理中的检查点只会有一个。
- number of concurrent checkpoints: 默认情况下， 在一个检查点未结束前不会开始另外一个检查点。 这个参数可以确保系统不会在检查点上花费太多的见时间，以影响正常的流处理。 可以允许重叠的检查点， 如果希望在特定的处理延迟情况下仍需要频繁地触发检查点，以便在故障时只要很少的再处理来说，还是挺有用的。 

    设置了`minimum time between checkpoints`的情况下不能使用这个设置。 

- externalized checkpoints： 可以配置定期的导出检查点。 导出检查点会将元数据写出到持久化存储， 作业失败时不会被清除。 这样就可以在作业失败时，从检查点恢复了。 详细参照[externalized-checkpoints](https://ci.apache.org/projects/flink/flink-docs-release-1.8/ops/state/checkpoints.html#externalized-checkpoints)

- fail/continue task on checkpoint errors: 决定了如果在任务的检查点处理失败时，任务是否失败。 默认是fail， 如果是continue,  任务会忽略检查点错误继续执行。

```java


StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// start a checkpoint every 1000 ms
env.enableCheckpointing(1000);

// advanced options:

// set mode to exactly-once (this is the default)
env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);

// make sure 500 ms of progress happen between checkpoints
env.getCheckpointConfig().setMinPauseBetweenCheckpoints(500);

// checkpoints have to complete within one minute, or are discarded
env.getCheckpointConfig().setCheckpointTimeout(60000);

// allow only one checkpoint to be in progress at the same time
env.getCheckpointConfig().setMaxConcurrentCheckpoints(1);

// enable externalized checkpoints which are retained after job cancellation
env.getCheckpointConfig().enableExternalizedCheckpoints(ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);


```
### Related Config Options
可以通过 conf/flink-conf.yaml 配置的参数 (详细参考[配置指南](https://ci.apache.org/projects/flink/flink-docs-release-1.8/ops/config.html)):
|   Key |   Default  |  Description  |
|    ---     |   ---     |   ---     |
|   state.backend    |  (none)   |  用来存储状态和检查点的状态后端  |
|   state.backend.async  |  TRUE     |  在可用的情况下，配置是否使用异步快照的方式。 有一些状态后端不支持异步快照，或只支持异步快照，会忽略这个配置。  |
|   state.backend.fs.memory-threshold    |  1024     |  状态数据文件的最小size。 所有小于这个值的状态块者以内联的方式存储在根检查点元数据文件内。  |
|   state.backend.incremental    |  FALSE    |  可用的情况下，是否开启增量检查点。 增量检查点只保存与前一个检查点的差异，而不是完成的检查点状态。 一些状态后端不支持增量检查点， 会忽略这个配置。   |
|   state.backend.local-recovery     |  FALSE    |  是否开始状态后端的本地恢复功能。  默认情况下是不开启 的。 现在本地恢复只支持有键的状态后端。 现在 MemoryStateBackend 不支持本地恢复，会忽略这个配置。  |
|   state.checkpoints.dir    |  (none)   |  Flink用来存放数据文件和元数据文件的默认位置。 这个位置必须能被所有的处理节点访问（TaskManagers and JobManagers）    |
|   state.checkpoints.num-retained   |  1    |  保留检查点的最大数目   |
|   state.savepoints.dir     |  (none)   |  保存点的默认目录。 状态后端用来存放保存点（MemoryStateBackend, FsStateBackend, RocksDBStateBackend）   |
|   taskmanager.state.local.root-dirs    |  (none)   |  本地恢复存放状态数据文件的根目录。  现在本地恢复只支持有键的状态后端。 现在 MemoryStateBackend 不支持本地恢复，会忽略这个配置。  |


## 选择状态后端

Flink 的 [检查点机制](https://ci.apache.org/projects/flink/flink-docs-release-1.8/internals/stream_checkpointing.html) 保存了计时器和有状态算子中所有状态的一致性快照，包括 connectors, windows, 和其它 [用户自定义状态](https://ci.apache.org/projects/flink/flink-docs-release-1.8/dev/stream/state/state.html) 。  检查点的存储位置 (e.g., JobManager memory, file system, database) 取决于状态后端的配置。

默认状态是保存在TaskManagers的内存中， 检查点保存在JobManager的内存中。 为了合适地持久化大状态， Flink在不同的状态后端中提供了多种处理方案来存储和checkpointing 状态。  可以通过**StreamExecutionEnvironment.setStateBackend(…)** 来选择状态后端。

更多的详细信息请参考[状态后端](https://ci.apache.org/projects/flink/flink-docs-release-1.8/ops/state/state_backends.html)



## 迭代作业中的状态检查点
Flink 现在只为非迭代的作业提供处理保证。 在迭代作业上启用检查点会导致异常。 为了在迭代的程序上强制执行检查点， 用户在启用检查点需要设置一个额外的标志： **env.enableCheckpointing(interval, CheckpointingMode.EXACTLY_ONCE, force = true)**。

要注意的是，还在循环中处理的数据和状态变更，在故障时会丢失。 


## 重启策略

Flink 支持不同的重启策略来控制在故障时作业如何重启， 详细参考[重启策略](https://ci.apache.org/projects/flink/flink-docs-release-1.8/dev/restart_strategies.html)。


# 可查询的状态
# 状态后端


# 演进更新状态模式
# 自定义状态的序列化