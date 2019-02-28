
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



Windows 是处理无限流的核心。 Windows 将流分割成有限大小的 buckets, 以在上面应用计算。 

下面是 Flink windowed 代码的一一般结构。 第一个是有键有， 第二个是无键的。 唯一的区别是有键的调用方式 为keyBy().window()， 无键的为 windowAll()。

####  Keyed Windows
stream
       .keyBy(...)               <-  keyed versus non-keyed windows
       .window(...)              <-  required: "assigner"
      [.trigger(...)]            <-  optional: "trigger" (else default trigger)
      [.evictor(...)]            <-  optional: "evictor" (else no evictor)
      [.allowedLateness(...)]    <-  optional: "lateness" (else zero)
      [.sideOutputLateData(...)] <-  optional: "output tag" (else no side output for late data)
       .reduce/aggregate/fold/apply()      <-  required: "function"
      [.getSideOutput(...)]      <-  optional: "output tag"

#### Non-Keyed Windows
stream
       .windowAll(...)           <-  required: "assigner"
      [.trigger(...)]            <-  optional: "trigger" (else default trigger)
      [.evictor(...)]            <-  optional: "evictor" (else no evictor)
      [.allowedLateness(...)]    <-  optional: "lateness" (else zero)
      [.sideOutputLateData(...)] <-  optional: "output tag" (else no side output for late data)
       .reduce/aggregate/fold/apply()      <-  required: "function"
      [.getSideOutput(...)]      <-  optional: "output tag"



## Windows Lifecycel   容器生命周期

简单地说， 当第一个元素到达时， 对应的window会立即被创建， 当时间（事件时间、处理时间）过了window 的终了时间+容许延时后，window会被完全的清理。 Flink 保证只清理基于时间的容器，并不清理其它类型的window,如 global windows。 

另外， 每一个 windows 都会有一个 Trigger 和 function (ProcessWindowFunction, ReduceFunction, AggregateFunction 或 FoldFunction)。 function 包含了将要应用在 window 内容上的计算逻辑。 Trigger 指定了window 什么时候应用function的条件。 条件可以是 “window 中元素的个数超过4个”， 或者 “水位过了 window 的终了时间”。 Trigger 还可以指定在是否清除 window 的内容。 这里的清除是仅指 windows中的元素， 而不是 window 本身的元信息， 也意味着，新到的数据依然可以添加到这个window。

除了上面的， 还可以指定一个 Evictor , 用于在 Trigger 触发之后， function 执行前或执行后 从window中移除元素。


## Keyed vs Non-Keyed Windows  有键与无键窗口
首先要指明的是你的流是有键的还是无键的。 这个必须在定义 window 前 确定下来。 使用了keyBy() 会将无限流分成逻辑上的有键流， 反之刚就是无键的流。

如果是有键流， 事件数据的任何属性都可以做为键。 有键流允许多任务并行的方式执行window 的计算逻辑， 如同每个键的数据是相互独立的。 所有的元素的键是一样的话， 将会被发送到同一个任务中处理。
无键流的话， 原始流则不会被分为多个逻辑上的流，（可以认认为所有的元素键值是一样的）


## window Assigners 窗口分配器
指定是否有键之后， 就是定义 window assigner。 窗口分配器定义了一个元素是如何分配到窗口的。 可以通过 keyedStream.window(...), 和 dataStream.windowAll(...) 来指定分配器。

分配器负责为每一个进来的元素指定一个或多个窗口。 Flink 预定义了一些常用的分配器， 滚动窗口、 滑动窗口、 会话窗口、 全局窗口。 也可以扩展 WindowAssinger 类来实现自定义窗口分配器。 除了全局窗口，内置的窗口分配器都是基于时间的（处理时间、事件时间）。

基于时间的窗口 有一个开始时间（包含）和一个结束时间（不包含）来定义窗口的大小。 


### Tumbling Windows 滚动窗口
滚动窗口将每一个元素分配到一个特定大小的窗口中。 滚动容器大小固定且容器间没有重叠。 

... todo

时间段可以用 Time.milliseconds(x), Time.seconds(x), Time.minutes(x) 等来指定。

如最后一个例子，滚动窗口分配器可以接收一个可选的 offset 参数来调整窗口的时间基准。 

 没有offset 时 小时的滚动窗口的时间基准是epoch, 得到的时间窗口是 1:00:00.000 - 1:59:59.999, 2:00:00.000 - 2:59:59.999 等. 如果指定了一个 15分钟的offset， 就会得到 1:15:00.000 - 2:14:59.999, 2:15:00.000 - 3:14:59.999 等时间窗口. offsets 一个重要用途就是调整窗口的时区，如北京时间就需要指定 offset = Time.hours(-8)。

### Sliding Windows 滑动窗口
滑动容器将元素分配到一组固定长度的窗口。 类似于滚动窗口， 滑动窗口的大小也是通过 window size 参数来配置。 有一个 window slide 参数用来控制滑动的创建频度。 因此如果 slide 小于 size， 滑动窗口间是可以有重叠的， 这时一个元素可能被分配到多个窗口中。


... todo

与滚动窗口一样， 滑动窗口也可以指定 offset 参数， 功能一样。


### Session Windows 会话窗口

会话窗口通过会话的活跃性将元素分组。 会话窗口没有重叠且也没有开始结束时间。 过了一个特定的时间间隔没有接收到数据的话就认为结束一个窗口。 会话窗口的的间隔可以静态配置或者通过 extractor function 动态指定。 超过间隔时间后， 当前的窗口关闭， 后续的数据元素会被分配给一个新的会话窗口。


... todo

可以通过 SessionWindowTimeGapExtractor 接口来实现动态间隔

！！！ 会话窗口没有固定的开始结束时间，滚动滑动窗口不同。 内部实现上，会话窗口是为每一个接收到的数据创建一个窗口， 并通过对比定义的会话间隔来将邻近的窗口合并。  因此， 会话窗口的 trigger 和 window Funciton 必须是可合并的。 可合并的 window function 如 ReduceFunction, AggregateFunction, ProcessWindowFunction。  （FoldFunction 是不能合并的）。


###  Global Windows 全局窗口
全局窗口将键值相同的元素全部分配到一个窗口中。 全局窗口一般用在你需要自定义trigger 的时候， 否则的话，由于全局窗口没有结束时间， 我们的计算逻辑不会触发执行。



## Window Functions

指定assigner之后，需要指定想要在窗口上执行的计算逻辑。 一旦系统判定一个窗口准备好被处理时， window function 的计算逻辑会应用到窗口的元素上。

window function 有 ReduceFunction, AggregateFunction, FoldFunction 和 ProcessWindowFunction 几种。 前两个可以增量地处理数据，相对高效。 ProcessWindowFunction 会得到一个包含了窗口所有元素数据的 Iterable 对象， 和当前窗口的一些元信息。

ProcessWindowFunction 在调用前需要为窗口缓冲所有的元素数据  相对低效一些。 可以将 ProcessWindowFunction 与 ReduceFunction、 AggregateFunction、 FoldFunction 结合起来使用，达到增量处理 并获取窗口元信息的效果。


### ReduceFunction
定义如何将两个元素归并为一个同类型的结果。 增量处理。

... todo

### AggregateFunction
AggregateFunction 就一个更一般化的 ReduceFunction。 需要指定三个类型， 输入类型（IN）， 累加器类型(ACC)和输出类型(OUT)。  AggregateFunction 有相应的方法用来 
1. 将一个元素合并到累加器
2. 初始化一个累加器
3. 合并两个累加器
4. 从累加器获取输出结果

与ReduceFunction一样， 增量处理。

... todo 


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