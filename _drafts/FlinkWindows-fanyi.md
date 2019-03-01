Windows 是处理无限流的核心概念。 Windows 将流分割成有限大小的 buckets, 以在上面应用计算。 

下面是 Flink windowed 代码的一一般结构（套路）。 第一个是有键有， 第二个是无键的。 唯一的区别是有键的调用方式 为keyBy().window()， 无键的为 windowAll()。

####  Keyed Windows
```
stream
       .keyBy(...)               <-  变成带键的流
       .window(...)              <-  必须： 窗口分配器 （assigner）
      [.trigger(...)]            <-  可选： 触发器 （assigner 有一个默认的）
      [.evictor(...)]            <-  可选： 驱逐器 （默认为 null）
      [.allowedLateness(...)]    <-  可选：容许延迟 (默认为 0)
      [.sideOutputLateData(...)] <-  可选：延时数据分支流标签 (默认没有延时数据分支流)
       .reduce/aggregate/fold/apply()      <-  必须: 用户自定义函数（UDF）
      [.getSideOutput(...)]      <-  可选: 获取分支流
```

#### Non-Keyed Windows
```
stream
       .windowAll(...)           <-  必须： 窗口分配器 （assigner）
      [.trigger(...)]            <-  可选： 触发器 （assigner 有一个默认的）
      [.evictor(...)]            <-  可选： 驱逐器 （默认为 null）
      [.allowedLateness(...)]    <-  可选：容许延迟 (默认为 0)
      [.sideOutputLateData(...)] <-  可选：延时数据分支流标签 (默认没有延时数据分支流)
       .reduce/aggregate/fold/apply()      <-  必须: 用户自定义函数（UDF）
      [.getSideOutput(...)]      <-  可选: 获取分支流
```


## Windows Lifecycel   窗口生命周期

简单地说， 当第一个元素到达时， 对应的窗口会立即被创建， 当时间（事件时间、处理时间）过了窗口 的结束时间+容许延时后，窗口会被完全的清理。 Flink 保证只清理基于时间的窗口，并不清理其它类型的window,如 global windows。 

另外， 每一个 窗口 都会有一个 Trigger 和 UDF (ProcessWindowFunction, ReduceFunction, AggregateFunction 或 FoldFunction)。 UDF 包含了将要应用在 window 内容上的计算逻辑。 Trigger 指定了用来判定window 什么时候应用UDF的条件。 这个条件可以是 “窗口中数据项的个数超过4个”， 或者 “水位（watermark）超过了 window 的结束时间”。 Trigger 还可以指定在是否清除 窗口 的内容数据。 这里的清除是仅指 windows中的数据项， 而不是 window 本身的元信息， 也就是说，后续到达的数据项依然是可以添加到这个window的。

除了上面这些， 还可以指定一个驱除器（Evictor）从window中移除数据项 , 移除的时机为 Trigger 触发之后， UDF 执行前或执行后（可配置） 。


## Keyed vs Non-Keyed Windows  有键与无键窗口
首先要明确你的流是有键的还是无键的。 这个必须在定义窗口前确定下来。 使用了keyBy() 在逻辑上会将一个无界流根据数据项的键值分成几个有键流（依然是无界的）， 不使用keyBy()则还是原始的无键流。

对于有键流， 事件数据的任何属性都可以做为键。 有键流允许多任务并行的方式执行window 的计算逻辑， 如同每个键的数据是相互独立的。 所有的元素的键是一样的话， 将会被发送到同一个任务中处理（译注： 逻辑上健值一样的数据项会分到一个独立的流中）。
无键流的话， 原始流则不会被分为多个逻辑上的流，（也可以认为所有的元素键值是一样的）


## window Assigners 窗口分配器
确定否有键之后， 就是定义 window assigner。 窗口分配器定义了一个元素是如何分配到窗口的。 可以通过 keyedStream.window(...), 和 dataStream.windowAll(...) 来指定分配器。

分配器负责为每一个进来的元素指定一个或多个窗口。 Flink 预定义了一些常用的分配器：， 滚动窗口、 滑动窗口、 会话窗口、 全局窗口。 也可以扩展 WindowAssinger 类来实现自定义窗口分配器。 除了全局窗口，内置的窗口分配器都是基于时间的（处理时间、事件时间）。

基于时间的窗口 有一个开始时间（包含）和一个结束时间（不包含）来定义窗口的大小。 

### Tumbling Windows 滚动窗口
滚动窗口将每一个元素分配到一个特定时间的窗口中。 滚动窗口时间固定且窗口间没有重叠。 
如下图所示， 可以指定一个大小为5分钟的滚动窗口，每5分钟新建一个窗口：

![滚动窗口](https://ci.apache.org/projects/flink/flink-docs-release-1.7/fig/tumbling-windows.svg)

代码示例：
```

DataStream<T> input = ...;

// tumbling event-time windows
input
    .keyBy(<key selector>)
    .window(TumblingEventTimeWindows.of(Time.seconds(5)))
    .<windowed transformation>(<window function>);

// tumbling processing-time windows
input
    .keyBy(<key selector>)
    .window(TumblingProcessingTimeWindows.of(Time.seconds(5)))
    .<windowed transformation>(<window function>);

// daily tumbling event-time windows offset by -8 hours.
input
    .keyBy(<key selector>)
    .window(TumblingEventTimeWindows.of(Time.days(1), Time.hours(-8)))
    .<windowed transformation>(<window function>);
```

时间段可以用 Time.milliseconds(x), Time.seconds(x), Time.minutes(x) 等来指定。

最后一个例子中，滚动窗口分配器传了一个可选的 offset 参数来调整窗口的时间基准。 

> 没有offset 时 小时的滚动窗口的时间基准是 epoch (1970-01-01 00:00:00 UTC), 得到的时间窗口是 1:00:00.000 - 1:59:59.999, 2:00:00.000 - 2:59:59.999 等. 如果指定了一个 15分钟的offset， 就会得到 1:15:00.000 - 2:14:59.999, 2:15:00.000 - 3:14:59.999 等时间窗口. offsets 一个重要用途就是调整窗口的时区，如想要用北京时间就需要指定 offset = Time.hours(-8)。

### Sliding Windows 滑动窗口
滑动容器将元素分配到一组固定长度的窗口。 类似于滚动窗口， 滑动窗口的大小也是通过 window size 参数来配置。 有一个 window slide 参数用来控制滑动的创建频度。 因此如果 slide 小于 size， 滑动窗口间是可以有重叠的， 这时一个元素可能被分配到多个窗口中。

如下图所示， 指定了一个长度为10分钟，滑动步长5分钟的滑动窗口，每5分钟新建一个长度为10分钟的时间窗口：

![滑动窗口](https://ci.apache.org/projects/flink/flink-docs-release-1.7/fig/sliding-windows.svg)

代码示例

```

DataStream<T> input = ...;

// sliding event-time windows
input
    .keyBy(<key selector>)
    .window(SlidingEventTimeWindows.of(Time.seconds(10), Time.seconds(5)))
    .<windowed transformation>(<window function>);

// sliding processing-time windows
input
    .keyBy(<key selector>)
    .window(SlidingProcessingTimeWindows.of(Time.seconds(10), Time.seconds(5)))
    .<windowed transformation>(<window function>);

// sliding processing-time windows offset by -8 hours
input
    .keyBy(<key selector>)
    .window(SlidingProcessingTimeWindows.of(Time.hours(12), Time.hours(1), Time.hours(-8)))
    .<windowed transformation>(<window function>);


```

与滚动窗口一样， 滑动窗口也可以指定 offset 参数， 功能一样。


### Session Windows 会话窗口

会话窗口根据数据的连续性（会话活跃性）将数据项进行分组。 会话窗口没有重叠且也没有开始结束时间。 如果一段时间里没有接收到任何数据的话就结束一个窗口。 会话窗口的的时间间隔可以静态配置或者通过 extractor function 动态指定。 超过指定的间隔时间后，当前的窗口关闭，后续到达的数据项会被分配给一个新的会话窗口。

![会话窗口](https://ci.apache.org/projects/flink/flink-docs-release-1.7/fig/session-windows.svg)

代码示例
```
DataStream<T> input = ...;

// event-time session windows with static gap
input
    .keyBy(<key selector>)
    .window(EventTimeSessionWindows.withGap(Time.minutes(10)))
    .<windowed transformation>(<window function>);
    
// event-time session windows with dynamic gap
input
    .keyBy(<key selector>)
    .window(EventTimeSessionWindows.withDynamicGap((element) -> {
        // determine and return session gap
    }))
    .<windowed transformation>(<window function>);

// processing-time session windows with static gap
input
    .keyBy(<key selector>)
    .window(ProcessingTimeSessionWindows.withGap(Time.minutes(10)))
    .<windowed transformation>(<window function>);
    
// processing-time session windows with dynamic gap
input
    .keyBy(<key selector>)
    .window(ProcessingTimeSessionWindows.withDynamicGap((element) -> {
        // determine and return session gap
    }))
    .<windowed transformation>(<window function>)
```



可以通过 SessionWindowTimeGapExtractor 接口来实现动态间隔

！！！ 与滚动滑动窗口不同，会话窗口没有固定的开始结束时间。在内部实现上，会话窗口分配器为每一个接收到的数据创建一个窗口。 后面通过对比定义的会话间隔来将邻近的窗口合并（时间差小于指定间隔的）。  因此， 会话窗口的 trigger 和 窗口UDF 必须是可合并的。 可合并的 窗口UDF 有 ReduceFunction, AggregateFunction, ProcessWindowFunction。  （FoldFunction 是不能合并的）。


###  Global Windows 全局窗口
全局窗口将键值相同的元素全部分配到一个窗口中。 全局窗口一般用在你需要自定义trigger 的时候， 否则的话，由于全局窗口没有结束时间， 我们的计算逻辑不会触发执行。

![全局窗口](https://ci.apache.org/projects/flink/flink-docs-release-1.7/fig/non-windowed.svg)

代码示例
```

DataStream<T> input = ...;

input
    .keyBy(<key selector>)
    .window(GlobalWindows.create())
    .<windowed transformation>(<window function>);

```

## Window Functions

指定assigner之后，需要通过UDF定义想要在窗口上执行的计算逻辑。 一旦系统判定一个窗口准备好被处理时， 窗口UDF 的计算逻辑会应用到窗口的数据项上。

窗口 UDF 有 ReduceFunction, AggregateFunction, FoldFunction 和 ProcessWindowFunction 几种。 前两个（译注： 应该是前三个吧， 原文有误？）可以在数据项分配到窗口时就增量地对数据项进行计算处理，相对高效。 ProcessWindowFunction 会得到一个包含了窗口所有元素数据的 Iterable 对象， 和当前窗口的一些元信息。

ProcessWindowFunction 是在调用前缓存窗口所有的数据项，窗口结束上触发计算，相对低效一些。 可以将 ProcessWindowFunction 与 ReduceFunction、 AggregateFunction、 FoldFunction 结合起来使用，达到增量处理并获取窗口元信息的效果。


### ReduceFunction
定义如何将两个数据项归并为一个类型相同的结果。 数据项分配到窗口时会增量地处理。

代码示例
```
DataStream<Tuple2<String, Long>> input = ...;

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .reduce(new ReduceFunction<Tuple2<String, Long>> {
      public Tuple2<String, Long> reduce(Tuple2<String, Long> v1, Tuple2<String, Long> v2) {
        return new Tuple2<>(v1.f0, v1.f1 + v2.f1);
      }
    });

```

### AggregateFunction
AggregateFunction 就一个更泛化的 ReduceFunction。 需要指定三个类型， 输入类型（IN）， 累加器类型(ACC)和输出类型(OUT)。  AggregateFunction 有相应的方法用来 
1. 将一个元素合并到累加器
2. 初始化一个累加器
3. 合并两个累加器
4. 从累加器获取输出结果

与ReduceFunction一样， 数据项分配到窗口时会增量地处理。

代码示例
```
/**
 * The accumulator is used to keep a running sum and a count. The {@code getResult} method
 * computes the average.
 */
private static class AverageAggregate
    implements AggregateFunction<Tuple2<String, Long>, Tuple2<Long, Long>, Double> {
  @Override
  public Tuple2<Long, Long> createAccumulator() {
    return new Tuple2<>(0L, 0L);
  }

  @Override
  public Tuple2<Long, Long> add(Tuple2<String, Long> value, Tuple2<Long, Long> accumulator) {
    return new Tuple2<>(accumulator.f0 + value.f1, accumulator.f1 + 1L);
  }

  @Override
  public Double getResult(Tuple2<Long, Long> accumulator) {
    return ((double) accumulator.f0) / accumulator.f1;
  }

  @Override
  public Tuple2<Long, Long> merge(Tuple2<Long, Long> a, Tuple2<Long, Long> b) {
    return new Tuple2<>(a.f0 + b.f0, a.f1 + b.f1);
  }
}

DataStream<Tuple2<String, Long>> input = ...;

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .aggregate(new AverageAggregate());

```


### FoldFunction


FoldFunction 定义如何把一个窗口的输入项合并到一个输入项上（与输入类型可心不一样）。 每一个数据项被分配到窗口时，FoldFunction 会被增量地调用，并把结果增量地合并到输出值上。窗口的第一个数据项会被合并到输出项的初始值上。

FoldFunction 可以这样用:

```
DataStream<Tuple2<String, Long>> input = ...;

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .fold("", new FoldFunction<Tuple2<String, Long>, String>> {
       public String fold(String acc, Tuple2<String, Long> value) {
         return acc + value.f1;
       }
    });
```

上面的例子将所有的 Long 类型的输入值都追加到一个初始为空的字符串。

注意： fold() 不能用于 session 窗口其它可合并的窗口。


### ProcessWindowFunction
A ProcessWindowFunction gets an Iterable containing all the elements of the window, and a Context object with access to time and state information, which enables it to provide more flexibility than other window functions. This comes at the cost of performance and resource consumption, because elements cannot be incrementally aggregated but instead need to be buffered internally until the window is considered ready for processing.

ProcessWindowFunction 长这样:
```
public abstract class ProcessWindowFunction<IN, OUT, KEY, W extends Window> implements Function {

    /**
     * Evaluates the window and outputs none or several elements.
     *
     * @param key The key for which this window is evaluated.
     * @param context The context in which the window is being evaluated.
     * @param elements The elements in the window being evaluated.
     * @param out A collector for emitting elements.
     *
     * @throws Exception The function may throw exceptions to fail the program and trigger recovery.
     */
    public abstract void process(
            KEY key,
            Context context,
            Iterable<IN> elements,
            Collector<OUT> out) throws Exception;

      /**
       * The context holding window metadata.
       */
      public abstract class Context implements java.io.Serializable {
          /**
           * Returns the window that is being evaluated.
           */
          public abstract W window();

          /** Returns the current processing time. */
          public abstract long currentProcessingTime();

          /** Returns the current event-time watermark. */
          public abstract long currentWatermark();

          /**
           * State accessor for per-key and per-window state.
           *
           * <p><b>NOTE:</b>If you use per-window state you have to ensure that you clean it up
           * by implementing {@link ProcessWindowFunction#clear(Context)}.
           */
          public abstract KeyedStateStore windowState();

          /**
           * State accessor for per-key global state.
           */
          public abstract KeyedStateStore globalState();
      }

}

```


Note The key parameter is the key that is extracted via the KeySelector that was specified for the keyBy() invocation. In case of tuple-index keys or string-field references this key type is always Tuple and you have to manually cast it to a tuple of the correct size to extract the key fields.


ProcessWindowFunction 的用法:
```


DataStream<Tuple2<String, Long>> input = ...;

input
  .keyBy(t -> t.f0)
  .timeWindow(Time.minutes(5))
  .process(new MyProcessWindowFunction());

/* ... */

public class MyProcessWindowFunction 
    extends ProcessWindowFunction<Tuple2<String, Long>, String, String, TimeWindow> {

  @Override
  public void process(String key, Context context, Iterable<Tuple2<String, Long>> input, Collector<String> out) {
    long count = 0;
    for (Tuple2<String, Long> in: input) {
      count++;
    }
    out.collect("Window: " + context.window() + "count: " + count);
  }
}


```

The example shows a ProcessWindowFunction that counts the elements in a window. In addition, the window function adds information about the window to the output.

Attention Note that using ProcessWindowFunction for simple aggregates such as count is quite inefficient. The next section shows how a ReduceFunction or AggregateFunction can be combined with a ProcessWindowFunction to get both incremental aggregation and the added information of a ProcessWindowFunction.

[todo]

### ProcessWindowFunction with Incremental Aggregation

A ProcessWindowFunction can be combined with either a ReduceFunction, an AggregateFunction, or a FoldFunction to incrementally aggregate elements as they arrive in the window. When the window is closed, the ProcessWindowFunction will be provided with the aggregated result. This allows it to incrementally compute windows while having access to the additional window meta information of the ProcessWindowFunction.

Note You can also the legacy WindowFunction instead of ProcessWindowFunction for incremental window aggregation.

##### Incremental Window Aggregation with ReduceFunction

The following example shows how an incremental ReduceFunction can be combined with a ProcessWindowFunction to return the smallest event in a window along with the start time of the window.

```
```


##### Incremental Window Aggregation with AggregateFunction

The following example shows how an incremental AggregateFunction can be combined with a ProcessWindowFunction to compute the average and also emit the key and window along with the average.

```
```


#### Incremental Window Aggregation with FoldFunction

The following example shows how an incremental FoldFunction can be combined with a ProcessWindowFunction to extract the number of events in the window and return also the key and end time of the window.

```
```




[todo]
### Using per-window state in ProcessWindowFunction

除了访问键状态（和其它rich function 一样）， ProcessWindowFunction 还可以使用当前窗口范围的键状态。 搞清楚 per-window state关联的窗口指的是什么非常重要。
窗口可能有不同的意思:

- 通过 windowed 算子指定的的窗口定义： 可以是1小时的滚动窗口， 也可以是一个长度为2 小时，滑动步长为1小时的滑动窗口
- 一个键值对应的一个窗口定义的一个实例： 可以是 userid为xyz的 [12:00 - 13:00 ) 时间窗口。 基于窗口的定义，根据作业当前正在处理的键的数量以及事件落入的时间段，将有许多窗口实例。


每窗口状态与后一个“窗口”相关联。也就是说，如果我们处理1000个不同键的事件，并且当前所有键的事件都属于[12:00，13:00)时间窗口，那么将有1000个窗口实例，每个窗口实例都有自己的键状态。

调用process()时， 得到的上下文对象有两个方法来访问两种状态:

- globalState(), 用来访问非窗口实例生命周期范围的键状态
- windowState(), 用来访问窗口实例生命周期范围的键状态

如果同一窗口实例会发生多次触发，则这个功能是非常有用。比如因为延时数据而发生延时触发，或者自定义的触发器会做一些预触发。这种情况下，可以将之前的触发信息或触发次数存储在窗口键状态中。

使用窗口状态时，清除窗口时也要清除该状态（重要），应该在clear（）方法中处理。 

### WindowFunction (Legacy)

在一些使用 ProcessWindowFunction 的地方也可以使用 WindowFunction。 WindowFunction 是一个老版本的 ProcessWindowFunction， 提供的上下文内容相对较少， 少了一些高级功能，如 per-window keyed state。 这个接口将来会废弃掉。

WindowFunction 长这样:

```
public interface WindowFunction<IN, OUT, KEY, W extends Window> extends Function, Serializable {

  /**
   * Evaluates the window and outputs none or several elements.
   *
   * @param key The key for which this window is evaluated.
   * @param window The window that is being evaluated.
   * @param input The elements in the window being evaluated.
   * @param out A collector for emitting elements.
   *
   * @throws Exception The function may throw exceptions to fail the program and trigger recovery.
   */
  void apply(KEY key, W window, Iterable<IN> input, Collector<OUT> out) throws Exception;
}

```

WindowFunction 可以这样用:
```
DataStream<Tuple2<String, Long>> input = ...;

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .apply(new MyWindowFunction());
```


## Triggers
触发器决定一个窗口什么时候可以被窗口function 处理。 每个 WindowAssigner 都有一个默认的触发器(Trigger)。 如果默认的触发器不满足需求， 可以通过 trigger(...) 方法指定一个自定义的触发器。

Trigger 接口有五个方法来处理不同的事件：
- onElement()： 每个元素数据被添加到本窗口时调用
- onEventTime(): 当注册的 evetn-time 定时器触发时调用
- onProcessingTime(): 当注册的 processing-time 定时器触发时调用
- onMerage(): 两个窗口合并时，用来合并两个有状态的tigger 的状态， 如 session 窗口
- clear(): 当容器被 PURGE 时用来清理trigger 持有的相关状态

注意： 
1. 前面三个方法返回一个 TriggerResult
      - CONTINUE: 什么都不做
      - FIRE: 触发窗口计算
      - PURGE: 清除窗口内容
      - FIRE_AND_PURGE: 触发计算并清除窗口内容
2. 每个方法都可以用来为未来动作注册一个定时器（基于 processing 或 event time ）


### Fire and Purge
如果触发认为窗口准备好应用计算逻辑时， 就通过返回 FIRE 或 FIRE_AND_PURGE 来触发， 这是当前窗口发送出窗口结果的信号。 对于 ProcessWindowFunction 的窗口， 所有的窗口数据会一起传递给 ProcessWindowFunction 处理；对于 ReduceFunction、 AggregateFunction 和 FoldFunction 的窗口， 只是将它们之前增量计算好的结果提取出来发送。


FIRE 会保留窗口的数据项内容。
FIRE_AND_PURGE 会将窗口的数据项内容清理掉，但不会清除窗口的元信息和trigger的状态。

### Default Triggers of WindowAssigners

所有基于事件时间的窗口分配器默认的触发器是 EventTimeTrigger， 一旦水位（watermark）超过窗口的结束时间就会触发处理。

注意： GlobalWindow 的默认触发器是 NeverTrigger - 从不触发。 使用 GlobalWindow 时，需要指定一个自定义触发器（译注：参考countWindow()）。

注意： 通过 trigger() 来指定自定义的 trigger 后， WindowAssigner 的默认 trigger 会被覆盖掉。 比如， 如果为TumblingEventTimeWindows 指定了一个 CountTrigger， 那么这个窗口的触发方式不再是基于事件的时间，而是基于数量了。 如果是一个基于时间+数据的需求，可以考虑自定义一个 trigger。 

### Built-in and Custom Triggers

Flink 的内置触发器

- EventTimeTrigger : 基于事件时间和水位来触发
- ProcessingTimeTrigger ： 基于处理时间（机器系统时间）来触发
- CountTrigger ： 基于数据项的数目来触发
- PurgingTrigger ： 封装其它的 trigger， 将 FIRE 转换为 FIRE_AND_PURGE

如果需要自定义 trigger , 需要理解一下 抽象类 Trigger 的 javaDOC， 需要注意的是在未来的版本中 Trigger 的API可能会迭代变更。


## Evictors

除了WindowAssigner 和 Trigger，Flink 还可以通过 evictor() 方法为窗口指定一个可选的 Evictor。 Evictor 可以在 trigger 触发后，调用UDF前/后，从窗口中移除数据项。 Evictor 有两个方法： 

```
/**
 * Optionally evicts elements. Called before windowing function.
 *
 * @param elements The elements currently in the pane.
 * @param size The current number of elements in the pane.
 * @param window The {@link Window}
 * @param evictorContext The context for the Evictor
 */
void evictBefore(Iterable<TimestampedValue<T>> elements, int size, W window, EvictorContext evictorContext);

/**
 * Optionally evicts elements. Called after windowing function.
 *
 * @param elements The elements currently in the pane.
 * @param size The current number of elements in the pane.
 * @param window The {@link Window}
 * @param evictorContext The context for the Evictor
 */
void evictAfter(Iterable<TimestampedValue<T>> elements, int size, W window, EvictorContext evictorContext);
```


evictBefore() 用来定义UDF调用前的数据项移除逻辑， evictAfter() 用来定义UDF调用后的数据项移除逻辑。 UDF调用前移除的数据项将不参与UDF的处理。


Flink 预定义的三个 Evictor:

- CountEvictor: 保留前面几个数据项（指定配置的数目）， 忽略后面的数据项。
- DeltaEvictor: 通过一个DeltaFunction计算窗口中每一个数据项与最后一个数据项的delta， 移除delta值大于等于指定阈值的数据项
- TimeEvictor: 指一个时间间隔 interval, 找到当前窗口数据项携带的最大的时间 max_time， 移除那些时间小于等于（max_time - interval）的数据项。


所有内置的 evictor 默认都是在UDF前调用的的， 可以配置为UDF后调用。

注意： 指定了Evictor ， Flink就不会为UDF进行预聚合的增量处理了， 所有的窗口数据项会缓存起来直到窗口触发后才进行计算逻辑的处理。

注意： Flink 不保证一个窗口内数据项的顺序。 意味着 Evictor 移除的数据项的顺序是不定的。 如 CountEvictor 保留的不是一定是最早到达窗口的几个数据项。


## Allowed Lateness

对于基于 Event-Time 的窗口时， 数据项可能会出现延迟到达： 数据到达时 用于跟踪度量事件时间的watermark已经超过该数据所属窗口的结束时间。

迟于水位（watermark）的到达的数据项默认是直接丢弃的。  Flink 可以为窗口操作指定一个 最大容许延迟，作为数据项不被丢弃的最大延迟时间，默认是 0 。 窗口结束之后，但在容许延迟时间前到达的数据项，依然可以被分配到该窗口。 根据使用的 trigger ，延迟但未被丢弃的数据项可以再次触发窗口的计算。 如 EventTimeTrigger。

为了达到这个目的， Flink会将窗口的状态保存到最大延迟也超时。  之后， Flink会删除窗口及容器的状态数据。

延迟时间默认是0。 也就是说延迟的数据都会被丢弃。

可以这样指定延迟时间：

```
DataStream<T> input = ...;

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .allowedLateness(<time>)
    .<windowed transformation>(<window function>);
```

如果使用的是 GlobalWindows 就没有延迟数据的概念了，因为 GlobalWindows 的结束时间是 Long.MAX_VALUE。


### Getting late data as a side output  打到侧输出流处理延迟数据
Using Flink’s side output feature you can get a stream of the data that was discarded as late.
通过Flink 的侧输出流的特性， 可以得到到一个被丢弃的延迟数据的流。


You first need to specify that you want to get late data using sideOutputLateData(OutputTag) on the windowed stream. Then, you can get the side-output stream on the result of the windowed operation:

首先通过 sideOutputLateData(OutputTag) 方法 在 windowStream 定义一个延迟数据的侧输出流。 然后， 可以在 窗口结果上取得这个侧输出流：

```

final OutputTag<T> lateOutputTag = new OutputTag<T>("late-data"){};

DataStream<T> input = ...;

SingleOutputStreamOperator<T> result = input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .allowedLateness(<time>)
    .sideOutputLateData(lateOutputTag)
    .<windowed transformation>(<window function>);

DataStream<T> lateStream = result.getSideOutput(lateOutputTag);

```

### Late elements consideratoins 关于延迟数据的思考

若指定了延迟时间， 窗口结束时间后， 窗口及窗口内容数据会保留到容许的延迟时间。 这时， 一个延迟但不应被丢弃的数据项到达时， 会再次触发窗口的计算逻辑，叫做延迟触发（第一次触发叫 主触发）。 对于 session 窗口来说， 延迟数据可能会引起已有窗口的合并（延迟数据的时间刚好连接了两个session）。

应该注意的是， 延迟触发的计算结果应该被视为前一次触发计算结果的更新。 也就是说对于一个计算，产生多次的结果。 程序中要注意这个重复结果的处理。


## Working with window results  使用窗口的结果数据

窗口操作的结果是一个 DataStream， 结果数据中没有窗口相关的信息。 如果想要保留窗口相关信息，需要自己在 ProcessWindowFunction 中手机设置到结果数据项中。 设置到结果数据项上唯一有用的信息就是窗口的时间了。 结果数据的时间(timestamp) 设置为最大容许时间（window.end_time -1, end_time 是不包含在窗口中的）。 event-Time 和 Processing-Time 是一样的。  

对于processing-time窗口，这没有特别的含义，但对于 event-time 窗口，加上水印与窗口交互，可以实现具有相同窗口大小的连续窗口操作。

下面会详细讨论到。

### Interaction of watermarks and windows

Before continuing in this section you might want to take a look at our section about event time and watermarks.

看下去前最好对 [事件时间与水位](https://ci.apache.org/projects/flink/flink-docs-release-1.7/dev/event_time.html)有个了解.

当水位到达窗口操作时会触发两个事情： 
- 触发所有结束时间-1 小于最新水位的窗口的计算处理
- 水位发送到下游操作


### Consecutive windowed operations 连续的窗口操作

As mentioned before, the way the timestamp of windowed results is computed and how watermarks interact with windows allows stringing together consecutive windowed operations. This can be useful when you want to do two consecutive windowed operations where you want to use different keys but still want elements from the same upstream window to end up in the same downstream window. Consider this example:



如前所述，窗口结果的时间戳的计算方式和水位与窗口的交互方式允许将连续的窗口操作串接起来。当想要执行两个连续的窗口化操作，且需要使用不同的键，而且想要下游窗口处理上游窗口的数据，则这一点非常有用。考虑这个例子： 

```
DataStream<Integer> input = ...;

DataStream<Integer> resultsPerKey = input
    .keyBy(<key selector>)
    .window(TumblingEventTimeWindows.of(Time.seconds(5)))
    .reduce(new Summer());

DataStream<Integer> globalResults = resultsPerKey
    .windowAll(TumblingEventTimeWindows.of(Time.seconds(5)))
    .process(new TopKWindowFunction());
```
在这个例子中， 第一个操作的时间窗口[0, 5)的结果会落到第二个操作时间窗口[0, 5)中。 第一个操计算[0, 5)中每个key 的总和， 第二个操作取得[0, 5)中总和数前几名（topk)的key。


## Useful state size considerations

可以定义非常长的时间窗口（几天、几周、几个月）， 但也会积累出体量非常大的状态数据。 估算窗口操作的存储需要时，有几个原则需要记住：

- Flink 会为每个窗口的每个数据项生成一份拷贝。 在滚动窗口中，每个数据项只有一份拷贝（一个数据项属于且只属于一个窗口，除非数据被丢弃）。 而在滑动窗口中， 每个数据项可能会有多个拷贝（一个数据项可能属于多个窗口）。所以，一个长度为一天，滑动间隔为一秒的滑动窗口绝逼不是一个好主意。 
- ReduceFunction, AggregateFunction, 和 FoldFunction 可以极大的减少存储需求，因为他们可以在数据到达时增量地计算合并数据项，一个窗口只需要保存一份数据值。 而 ProcessWindowFunction 则需要堆积所有的数据项。
- 使用 Evictor 的话，会阻止所有的增量处理， 因为在应用计算逻辑前，所有的数据项需要通过 evictor 来决定是参与计算。
