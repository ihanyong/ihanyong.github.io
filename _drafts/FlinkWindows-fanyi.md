
## Windows Lifecycel   容器生命周期

## Keyed vs Non-Keyed Windows  有键与无键窗口

## window Assigners 容器分配器
### Tumbling Windows 滚动窗口
### Sliding Windows 滑动窗口
### Session Windows 会话窗口
###  Global Windows 全局窗口


## Window Functions
### ReduceFunction
### AggregateFunction
### FoldFunction
### ProcessWindowFunction
### ProcessWindowFunction with Incremental Aggregation
### Using per-window state in ProcessWindowFunction
### WindowFunction (Legacy)

## Triggers
### Fire and Purge
### Default Triggers of WindowAssigners
### Built-in and Custom Triggers


## Evictors
## Allowed Lateness
### Getting late data as a side output
### Late elements consideratoins
## Working with window results
### Interaction of watermarks and windows
### Consecutive windowed operations
## Useful state size considerations
----------------------------------------------------------



Windows 是处理无限流的核心概念。 Windows 将流分割成有限大小的 buckets, 以在上面应用计算。 

下面是 Flink windowed 代码的一一般结构（套路）。 第一个是有键有， 第二个是无键的。 唯一的区别是有键的调用方式 为keyBy().window()， 无键的为 windowAll()。

####  Keyed Windows
stream
       .keyBy(...)               <-  变成带键的流
       .window(...)              <-  必须： 窗口分配器 （assigner）
      [.trigger(...)]            <-  可选： 触发器 （assigner 有一个默认的）
      [.evictor(...)]            <-  可选： 驱逐器 （默认为 null）
      [.allowedLateness(...)]    <-  可选：容许延迟 (默认为 0)
      [.sideOutputLateData(...)] <-  可选：延时数据分支流标签 (默认没有延时数据分支流)
       .reduce/aggregate/fold/apply()      <-  必须: 用户自定义函数（UDF）
      [.getSideOutput(...)]      <-  可选: 获取分支流

#### Non-Keyed Windows
stream
       .windowAll(...)           <-  必须： 窗口分配器 （assigner）
      [.trigger(...)]            <-  可选： 触发器 （assigner 有一个默认的）
      [.evictor(...)]            <-  可选： 驱逐器 （默认为 null）
      [.allowedLateness(...)]    <-  可选：容许延迟 (默认为 0)
      [.sideOutputLateData(...)] <-  可选：延时数据分支流标签 (默认没有延时数据分支流)
       .reduce/aggregate/fold/apply()      <-  必须: 用户自定义函数（UDF）
      [.getSideOutput(...)]      <-  可选: 获取分支流



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

... todo

时间段可以用 Time.milliseconds(x), Time.seconds(x), Time.minutes(x) 等来指定。

最后一个例子中，滚动窗口分配器传了一个可选的 offset 参数来调整窗口的时间基准。 

> 没有offset 时 小时的滚动窗口的时间基准是 epoch (1970-01-01 00:00:00 UTC), 得到的时间窗口是 1:00:00.000 - 1:59:59.999, 2:00:00.000 - 2:59:59.999 等. 如果指定了一个 15分钟的offset， 就会得到 1:15:00.000 - 2:14:59.999, 2:15:00.000 - 3:14:59.999 等时间窗口. offsets 一个重要用途就是调整窗口的时区，如想要用北京时间就需要指定 offset = Time.hours(-8)。

### Sliding Windows 滑动窗口
滑动容器将元素分配到一组固定长度的窗口。 类似于滚动窗口， 滑动窗口的大小也是通过 window size 参数来配置。 有一个 window slide 参数用来控制滑动的创建频度。 因此如果 slide 小于 size， 滑动窗口间是可以有重叠的， 这时一个元素可能被分配到多个窗口中。


... todo

与滚动窗口一样， 滑动窗口也可以指定 offset 参数， 功能一样。


### Session Windows 会话窗口

会话窗口根据数据的连续性（会话活跃性）将数据项进行分组。 会话窗口没有重叠且也没有开始结束时间。 如果一段时间里没有接收到任何数据的话就结束一个窗口。 会话窗口的的时间间隔可以静态配置或者通过 extractor function 动态指定。 超过指定的间隔时间后，当前的窗口关闭，后续到达的数据项会被分配给一个新的会话窗口。

... todo

可以通过 SessionWindowTimeGapExtractor 接口来实现动态间隔

！！！ 与滚动滑动窗口不同，会话窗口没有固定的开始结束时间。在内部实现上，会话窗口分配器为每一个接收到的数据创建一个窗口。 后面通过对比定义的会话间隔来将邻近的窗口合并（时间差小于指定间隔的）。  因此， 会话窗口的 trigger 和 窗口UDF 必须是可合并的。 可合并的 窗口UDF 有 ReduceFunction, AggregateFunction, ProcessWindowFunction。  （FoldFunction 是不能合并的）。


###  Global Windows 全局窗口
全局窗口将键值相同的元素全部分配到一个窗口中。 全局窗口一般用在你需要自定义trigger 的时候， 否则的话，由于全局窗口没有结束时间， 我们的计算逻辑不会触发执行。



## Window Functions

指定assigner之后，需要通过UDF定义想要在窗口上执行的计算逻辑。 一旦系统判定一个窗口准备好被处理时， 窗口UDF 的计算逻辑会应用到窗口的数据项上。

窗口 UDF 有 ReduceFunction, AggregateFunction, FoldFunction 和 ProcessWindowFunction 几种。 前两个（译注： 应该是前三个吧， 原文有误？）可以在数据项分配到窗口时就增量地对数据项进行计算处理，相对高效。 ProcessWindowFunction 会得到一个包含了窗口所有元素数据的 Iterable 对象， 和当前窗口的一些元信息。

ProcessWindowFunction 是在调用前缓存窗口所有的数据项，窗口结束上触发计算，相对低效一些。 可以将 ProcessWindowFunction 与 ReduceFunction、 AggregateFunction、 FoldFunction 结合起来使用，达到增量处理并获取窗口元信息的效果。


### ReduceFunction
定义如何将两个数据项归并为一个类型相同的结果。 数据项分配到窗口时会增量地处理。

... todo

### AggregateFunction
AggregateFunction 就一个更泛化的 ReduceFunction。 需要指定三个类型， 输入类型（IN）， 累加器类型(ACC)和输出类型(OUT)。  AggregateFunction 有相应的方法用来 
1. 将一个元素合并到累加器
2. 初始化一个累加器
3. 合并两个累加器
4. 从累加器获取输出结果

与ReduceFunction一样， 数据项分配到窗口时会增量地处理。

... todo 


### FoldFunction
### ProcessWindowFunction
### ProcessWindowFunction with Incremental Aggregation
### Using per-window state in ProcessWindowFunction
### WindowFunction (Legacy)

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

- CountEvictor: 
- DeltaEvictor: 
- TimeEvictor: 

    CountEvictor: keeps up to a user-specified number of elements from the window and discards the remaining ones from the beginning of the window buffer.
    DeltaEvictor: takes a DeltaFunction and a threshold, computes the delta between the last element in the window buffer and each of the remaining ones, and removes the ones with a delta greater or equal to the threshold.
    TimeEvictor: takes as argument an interval in milliseconds and for a given window, it finds the maximum timestamp max_ts among its elements and removes all the elements with timestamps smaller than max_ts - interval.

Default By default, all the pre-implemented evictors apply their logic before the window function.

Attention Specifying an evictor prevents any pre-aggregation, as all the elements of a window have to be passed to the evictor before applying the computation.

Attention Flink provides no guarantees about the order of the elements within a window. This implies that although an evictor may remove elements from the beginning of the window, these are not necessarily the ones that arrive first or last.



## Allowed Lateness
### Getting late data as a side output
### Late elements consideratoins
## Working with window results
### Interaction of watermarks and windows
### Consecutive windowed operations
## Useful state size considerations