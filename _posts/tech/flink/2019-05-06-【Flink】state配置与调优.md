---
layout: post
title:  "2019-05-06-【Flink】state配置与调优"
date:   2019-05-06 11:00:00 +0800
tags:
        - 流处理
        - Flink
---

参考
-  [Checkpoints](https://ci.apache.org/projects/flink/flink-docs-release-1.8/ops/state/checkpoints.html)
-  [Savepoints](https://ci.apache.org/projects/flink/flink-docs-release-1.8/ops/state/savepoints.html])
-  [State Backends](https://ci.apache.org/projects/flink/flink-docs-release-1.8/ops/state/state_backends.html)
-  [Tuning Checkpoints and Large State](https://ci.apache.org/projects/flink/flink-docs-release-1.8/ops/state/large_state_tuning.html)

# CheckPoint

通过保存状态与流的位置信息，使得Flink中的状态实现容错。

检查点只是用来恢复Flink作业的，当作业被取消后会被删除，默认不保留的，但可能通过配置将检查点保留一段时间：
- ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION: 取消作业后保留检查点， 需要删除的时候只能手动删除检查点。
- ExternalizedCheckpointCleanup.DELETE_ON_CANCELLATION: 取消作业后删除检查点，  查检点作业出现错误用于恢复作业时可用。


与保存点类似， 检查点由一个元数据文件和一些数据文件(取决于state backend)组成。 文件的保存路径由配置文件中的 **state.checkpoints.dir** 指定， 也可以在 作业的代码中指定。 

通过配置文件进行全局配置
```
state.checkpoints.dir: hdfs:///checkpoints/
```
通过代码为作业单独配置
```java
env.setStateBackend(new RocksDBStateBackend("hdfs:///checkpoints-data/");
```

与保存点的区别
- 使用特定的数据格式（底层）， 可以是增量的。
- 不支持如容量扩展等功能。


与保存点类似，作业可以使用查检点的元数据文件从检查点恢复。 如果元数据文件不是自包含的，jobmanager 需要可以访问到其指定的数据文件。
```
$ bin/flink run -s :checkpointMetaDataPath [:runArgs]
```

# SavePoints

保存点是一个流处理作业的运行时状态的一致性的镜像， 是通过Flink的检查点机制创建的。 可能通过保存点来停止-重启， 派生， 升级作业。  保存点由两个部分组成： 状态存储器上的二进制数据文件的文件夹（一般很大），和一个元数据文件（相对很小）。  状态存储器上的文件是作业运行时状态的镜像数据。 元数据主要是保存了保存点所有数据文件的绝对路径。

模仿上， 保存点与检查点的不同有点类似传统数据库的备份与恢复日志的区别。 检查点的意图是提供一个恢复机制防止意外的作业失败。 检查点的生命周期是由Flink管理的， 如创建，所属， 释放等， 不需要用户进行操作。 作为一个恢复的手段，和定期触发的操作，检查点有两个设计目标： 1） 创建需要是轻量级的， 2） 恢复尽可能的快速。   为了达成这些目标， 需要一些指定的性质，如不同的执行尝试间不改变作业的代码。 一般当用户停止作业后，检查点会被丢弃（可通过配置来保留检查点）。

与检查点相反， 保存点是由用户来创建，使用，删除的。  其主要用于有计划的手机备份与重启。 如， 可以用来升级Flink版本， 修改作业图， 修改并行度， 派生出第二个作业（红蓝发布） 等。  保存点在作业停止时是保留的。 概念上， 保存点的创建与恢复相对有点昂贵， 主要聚焦于可移动和支持前面提到的对作业的修改上。 


除了概念上的是不同， 当前检查点与保存点在实现上基本使用相同的代码和数据格式。 但是当前有现代战争例外， 而且未来也会引入更多的差异。 这个例外就是 RocksDB 状态后端的 增量检查点。 其使用了一些RocksDB 内存的数据格式来替代Flink的保存点格式， 使得检查点更加的轻量。

强烈建议通过udi(String）方法为算子指定ID ， 以便之后升级作业程序。 这个ID用于区分不同算子的状态。

```java
DataStream<String> stream = env.
  // 指定ID的有状态的源 (如. Kafka) 
  .addSource(new StatefulSource())
  .uid("source-id") // 源算子的ID
  .shuffle()
  // 指定ID的有状态的 mapper
  .map(new StatefulMapper())
  .uid("mapper-id") // mapper算子的ID
  // 无状态的 sink
  .print(); // 自动生成 ID
```

如果不指定ID， Flink会为算子自动生成一个ID。 如果ID不变的话， 可以自动地从保存点恢复出来。 自动生成的ID依赖于程序的结构，且对于程序的修改很敏感。 所以强烈建议为算子人为指定ID。

可以认为保存点为有状态的算子维护了一个（算子ID -> State） 的map。 
```
Operator ID | State
------------+------------------------
source-id   | State of StatefulSource
mapper-id   | State of StatefulMapper
```
上面的例子中， 打印sink 是无状态的，所能不是保存点状态的一部分。 默认地， Flink 会尝试将保存点的每一个条目映射回新的程序中。 

可以使用 命令行来触发保存点、取消作业时并生成保存点、从保存点重启作业、处置保存点。

在Flink >= 1.2.0 之后， 可以通过web页面从保存点重启作业。 


触发保存点时， 会创建一个新的保存点文件夹用来保存元数据文件和数据文件。 文件来做的位置可以通过 配置文件来配置默认的目标文件夹，也可以通过触发命令指定一个目标文件夹。 

目标文件夹必须是在一个JobManager(s) and TaskManager(s) 都能访问的分布式文件系统上。 

如 FsStateBackend 和 RocksDBStateBackend。

```
# 保存点目标目录
/savepoints/

# 保存点目录
/savepoints/savepoints-:shortjobid-:savepointid/

# 保存点元数据文件
/savepoints/savepoints-:shortjobid-:savepointis/_metadata

# 保存点状态
/savepoints/savepoints-:shortjobid-:savepointid/...
```

因为元数据中保存是的绝对路径，所以现在保存点的数据文件是不能移动的。  这个限制正在通过这个[LINK-5778](https://issues.apache.org/jira/browse/FLINK-5778) 进行解决。

如果使用的是MemoryStateBackend ， 元数据和保存点状态都是保存在 _metadata 文件中的， 因为是一个自包含的文件，可以将文件移动到任意的位置并加载恢复。

不建议移动或删除运行中的作业的最后一个保存点。 因为可能会影响容错恢复机制。 保存点对 exactly-once sink 是有副作用的：为了保证　 exactly-once　的语义，　如果在最后一次savepoint 之后没有生成检查点，会使用savepoint来进行恢复。

触发保存点
```
$ bin/flink savepoint :jobId [:targetDirectory]
```


触发YARN上作业的保存点
```
$ bin/flink savepoint :jobId [:targetDirectory] -yid :yarnAppId
```

取消作业时生成保存点
```
$ bin/flink cancel -s [:targetDirectory ] :jobId
```
从保存点重启
```
# 指定保存点的目录路径或 _metadata文件路径
$ bin/flink run -s :savepointPath [:runArgs]
```

忽略状态
从保存点重启时，默认会从保存点映射所有的状态到程序中的算子。 如果在程序中删除了算子，可能通过 --allowNonRestoredState(-n) 来允许新程序忽略删除的算子状态：
```
$ bin/flink run -s :savepointPath -n  [:runArgs]
```

处置保存点
```
$ bin/flink savepoint -d :savepointPath
```

也可以通过常规的文件文件系统操作来删除保存点，而不影响其它的保存点或检查点。 


可以在通过 **state.savepoints.dir** 配置一个默认的保存点目标目录。 触发保存点时， 这个目录用来存储保存点数据。 可以在触发命令中指定一个自定义的目标目录。
```
state.savepoints.dir: hdfs:///flink/savepoints
```

如果没有配置默认目标路径，也没有在命令行中指定目标路径， 保存点人触发会失败。


## F.A.Q

#### 需要为所有的算子指定id吗？ 
经验之谈， 是的。 严格来讲， 只为有状态的算子指定id是足够的了。 保存点只为算子保存状态， 无状态的算子不在保存点中。  

在实践上， 为所有的算子指定id， 因为一些内置的算子可能也是有状态的，如窗口算子。 哪些内置算子是有状态哪些是无状态的并不能明显的区分出来。 如果能绝对确定某个算子是无状态的，可以省略掉 uid() 方法。

#### 如果为作业增加了一个新的算子会怎样？ 
如果新增一个算子到作业中， 初始化时没有任务的状态。 保存点包含了所有有状态算子的状态。 无状态算子不在保存点中。 新增的算子，可以认为和无状态算子一样。

#### 如果从作业中删除一个有状态的算子会怎样？
默认情况下，从保存点重启时，会将保存点中所有的状态映射回作业中去。 如果保存点的的状态在作业中对应的算子被删除掉了， 作业启动会失败。 

可能在运行命令时指定 **--allowNonRestoredState (short: -n)** 来允许忽略被删除算子的状态：
```
$ bin/flink run -s :savepointPath -n [:runArgs]
```

#### 如果调整了作业中算子的顺序会怎样？
如果对算子指定了ID， 状态会正常加载回对应的算子。

如果没有指定算子的ID， 极有可能会造成有状态算子的自动生成的ID发生改变，最终造成无法从保存点加载状态。

#### 如果新增、删除、重排序作业中无状态的算子会怎样？

如果对有状态的算子都指定了ID， 那么无状态算子的修改对保存点的重加载没有影响。

如果没有对有状态的算子指定ID， 那么修改了无状态的算子可能会造成有状态算子的自动生成的ID发生改变，最终造成无法从保存点加载状态。


#### 重启时改变了作业程序的并行度会怎样？
如果保存点是  Flink >= 1.2.0 生成的， 且没有使用弃用的 state API 如 Checkpointed， 可以简单地从保存点重启程序并指定一个新的并行度。

如果使用了  Flink < 1.2.0  的保存点，或者使用了弃用的 APIs， 在修改并行度之前必须将作业和保存点都升级到  Flink >= 1.2.0。


#### 可以删除存储器上的保存点文件吗？
当前，不可以！  因为现在技术实现上，保存点的元数据文件里保存的是数据文件的绝对路径。 如果一定需要移动某些地文件， 有两个方法可以办到。 一、 简单但危险的： 修改元数据文件中的数据文件地址路径。 二、使用 SavepointV2Serializer 类作为切入点，在程序中读取维护重写元数据文件。


# 状态后端
Flink 提供了三种状态后端
- MemoryStateBackend  （默认）
- FsStateBackend
- RocksDBStateBackend

### MemoryStateBackend
RocksDBStateBackend 内部实现上，以Java堆内存对象的形式保存数据。 key/value 状态和容器算子维护一个hash表来保存值、触发器等。

对于检查点， RocksDBStateBackend 会快照状态，并将快照作为 检查点确认信息的一部分发送给 JobManager（Master），JobManager也是将其保存在堆内存中。

MemoryStateBackend  可以配置为异步快照。 为了避免阻塞，强烈建议使用异步快照， 当前的默认是开启异步快照的。 要关闭异步快照，需要在实例化 MemoryStateBackend 时 ，在构造函数中指定（应该只在调度时使用）：
```java
    new MemoryStateBackend(MAX_MEM_STATE_SIZE, false);
```
MemoryStateBackend 的限制：
- 默认单独的状态的大小最大为5M。 可以通过 MemoryStateBackend 的构造函数的增加这个值。
- 不考虑配置的配置的状态最大大小， 状态大小不能超过 akka的一帧。[参考Configuration](https://ci.apache.org/projects/flink/flink-docs-release-1.8/ops/config.html)
- 合计状态必须能放入 JobManager 内存。

MemoryStateBackend  建议使用场景:
- 本地开发与调试
- 状态很小的作业， 和只包含一次一事件的函数的作业（Map, FlatMap, Filter, ...）。 Kafka Consumer 只需要很小的状态。

### FsStateBackend
FsStateBackend 可以配置为一个文件系统的URL，如， “hdfs://namenode:40010/flink/checkpoints”，“file:///data/flink/checkpoints”。

FsStateBackend 在 TaskManager 的内存中保存即时数据， 在生成检查点时， 将状态快照写入到配置的文件目录下的文件中。 少量的元数据保存在 JobManager的内存中（HA模式下，保存在 元数据检查点中）。

FsStateBackend 默认是快照，以避免在写状态查检点时阻塞流处理。 可以在实例化 FsStateBackend 时禁用异步：

```java
    new FsStateBackend(path, false);
```

FsStateBackend 建议使用场景：
- 大体积的状态、 长窗口、 大体积的key/value 状态的作业。
- 所有的HA的设置

### RocksDBStateBackend
RocksDBStateBackend 使用 文件系统URL来配置，如 “hdfs://namenode:40010/flink/checkpoints”和“file:///data/flink/checkpoints”。

RocksDBStateBackend  在 TaskManager 本地的RocksDB数据库中保存即时数据， 在生成检查点时， 将整个RocksDB 数据库写入到配置的文件目录下。 少量的元数据保存在 JobManager的内存中（HA模式下，保存在 元数据检查点中）。

RocksDBStateBackend  只有异步模式。

RocksDBStateBackend 的限制：
- RocksDB 的JNI桥接API是基于 byte[]的， 每个key和每个value 最大支持 2^31 bytes。 ****IMPORTANT**： 在RocksDB中可合并的状态会静默的规程 大于 2^31 bytes 的值， 在下次取回的时候会失败。


RocksDBStateBackend 的建议使用场景：
- 非常大的状态、长窗口和大 key/value 状态的 作业。
- 所有的HA的设置

在 RocksDBStateBackend 中，能保存的状态大小只受限于磁盘空间。 相比于将状态保存在内存中的FsStateBackend而言， RocksDBStateBackend 可以保存非常大的状态。 也意味着最大吞吐量比较低。 所能对 RocksDB状态后端的读写，都要经过序列化与返序列化， 相比于基于堆内存的状态后端，这个操作更加的昂贵。

RocksDBStateBackend 是当前唯一一个支持增量检查点的状态后端。

一些RocksDB 特有的度量指标可以使用，但是默认是禁用的，可以参考的[文档](https://ci.apache.org/projects/flink/flink-docs-release-1.8/ops/config.html#rocksdb-native-metrics)




### 配置状态后端
如果没有指定，默认的状态后端就是jobmanager。 如果想为集群上所有的作业指定其它的默认状态后端， 可以在 flink-conf.yaml 进行配置。 每一个作业也可以覆盖默认的状态后端配置。

#### 为作业单独配置状态后端
作业级别的状态后端在 StreamExecutionEnvironment 上设置：

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setStateBackend(new FsStateBackend("hdfs://namenode:40010/flink/checkpoints"));

```

要使用 RocksDBStateBackend, 需要在 Flink的工程中添加下面依赖：
```xml
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-statebackend-rocksdb_2.11</artifactId>
    <version>1.8.0</version>
</dependency>
```

#### 配置默认状态后端

可在flink-conf.yaml 中 使用 **state.backend** 配置默认的状态后端。 

可配置的值是  jobmanager (MemoryStateBackend), filesystem (FsStateBackend), rocksdb (RocksDBStateBackend) 或者状态后端工厂StateBackendFactory实现类的全名， 如 org.apache.flink.contrib.streaming.state.RocksDBStateBackendFactory。

state.checkpoints.dir 定义了存放检查点数据和元数据文件的目录。  [详细可参考](https://ci.apache.org/projects/flink/flink-docs-release-1.8/ops/state/checkpoints.html#directory-structure)


配置示例：
```
# The backend that will be used to store operator state checkpoints
state.backend: filesystem


# Directory for storing checkpoints
state.checkpoints.dir: hdfs://namenode:40010/flink/checkpoints
```

| key | Default | 备注 |
| --- | --- | --- |
| state.backend.rocksdb.checkpoint.transfer.thread.num | 1 | 传输文件（上载下载）的线程数 |
| state.backend.rocksdb.localdir | (none) | RocksDB存放文件的本地文件夹（TaskManager） |
| state.backend.rocksdb.options-factory | "org.apache.flink.contrib.streaming.state.DefaultConfigurableOptionsFactory" | RocksDB  创建DBOptions  和 ColumnFamilyOptions 的工厂类。  |
| state.backend.rocksdb.predefined-options | "DEFAULT" | 预置的 RocksDB DBOptions 和 ColumnFamilyOptions。 当前支持的可选项 DEFAULT, SPINNING_DISK_OPTIMIZED, SPINNING_DISK_OPTIMIZED_HIGH_MEM or FLASH_SSD_OPTIMIZED 。 用户在OptionsFactory  自定义的选项优先于预定义的选项。|
| state.backend.rocksdb.timer-service.factory | "HEAP" |  HEAP (基于堆内存，默认) or ROCKSDB。 时间服务状态的实现的工厂。  |
| state.backend.rocksdb.ttl.compaction.filter.enabled | false | 是否启用TTL压缩过滤器来清除状态 |


# 调整检查点与大状态
为了保证在大规模的场景下Flink能可靠地运行，两个条件需要满足：
- 可靠的检查点
- 在发生故障时，资源足够堆积输入的数据流。

下面先讨论如何在大规模上获得性能良好的检查点。 然后解释一些容量规划的最佳实践。

## 监控状态和检查点
监控 检查点最简单的方式是通过UI的检查点页面。 详细参考[检查点监控](https://ci.apache.org/projects/flink/flink-docs-release-1.8/monitoring/checkpoint_monitoring.html)

在扩展检查点时特别有意思的两个数字：
- 开始检查点到算子的时间： 这个时间现在是不直接显示的，可以通过下面公式计算：
    checkpoint_start_delay = end_to_end_duration - synchronous_duration - asynchronous_duration
    如果这个时间一直很高的话， 说明检查点栅栏从source到当前算子需要很长时间。一般这表示当前系统处于一个持续的反压状态。
- 对齐时的数据缓冲量。 对于 exactly-once 语义， Flink会在接收多个输入流的算子上通过缓冲数据将流数据对齐。 理想状态下缓冲的数据应该非常的少， 如果缓冲很多的数据说明不同输入流上接收到的检查点栅栏时间差异较大。

这两个值偶尔很高，说明出现了背压、数据倾斜或者网络问题。 如果值一直很高， 说明Flink投入我过多的资源到检查点上。

## 调整检查点
检查点以规律的间隔定期触发， 应用中可以配置这个间隔。 如果检查点完成的时间长于触发间隔，那么在当前检查点完成之前下一检查点不会触发。默认情况，前一次检查点完成后下一次检查点会立即触发。 

当检查点的完成时间频繁地比配置间隔时间长时（）， 系统会一直处于检查点的处理之中。 这意味着过多的资源被检查点占用， 算子操作推进较慢。 异步检查点状态的配置下，这种行为对流处理影响有限， 但对整休的应用性能还是有一定影响的。

为了防止这种情况， 应用可以配置一个检查点间的最小时间段：
```java
StreamExecutionEnvironment.getCheckpointConfig().setMinPauseBetweenCheckpoints(milliseconds)
```
这个时间段是上次检查点完成后，到下次检查点开始前的最小时间间隔， 可参考下图：

![checkpoint_tuning](https://ci.apache.org/projects/flink/flink-docs-release-1.8/fig/checkpoint_tuning.svg)

应用可以配置允许多个检查点同时处理（通过CheckpointConfig）。 对于大状态的应用， 这样做经常会占用过多的资源到检查点的处理中。 当手动触发保存点时， 保存点的处理可能会和检查点并发执行。


## 调整网络缓冲区

Before Flink 1.3, an increased number of network buffers also caused increased checkpointing times since keeping more in-flight data meant that checkpoint barriers got delayed. Since Flink 1.3, the number of network buffers used per outgoing/incoming channel is limited and thus network buffers may be configured without affecting checkpoint times 。 

Flink 1.3之前，网络缓冲区的增加，也会引起检查点处理时间的增加。 因为需要保存更多的即时数据，意味着检查点栅栏被延迟。  Flink 1.3之后，每个输入输出的channel 使用的网络缓冲区是有限的， 因此网络缓冲区可以在不影响检查点时间的情况进行配置。  参考[network buffer configuration](https://ci.apache.org/projects/flink/flink-docs-release-1.8/ops/config.html#configuring-the-network-buffers).


## 尽可能使用异步检查点
异步快照的检查点，扩展性要比同步快照好很多。 尤其是在复杂的流应用中（多个jion， co-function, 窗口）， 有着深远的影响。

为了使用异步快照， 应用程序需要：
- 使用Flink管理的状态： 被管理的状态是指使用Flink提供的数据结构进行状态的保存。当前， 对于 keyed state 是支持的，使用接口进行抽象如 ValueState, ListState, ReducingState ... 
- 使用支持异步快照的状态后端： 在Flink 1.2， 只有 RocksDB 状态后端是完全支持异步快照的。 从 Flink 1.3 开始， 基于堆内存的状态后端也支持异步快照。 

上面两点也暗示着对于大状态一般应该保存为keyed state， 而不是算子状态。


## 调整 RocksDB
许多大型FLink流应用的状态存储后端是RocksDB。 这个状态后端的规模远超主内存，并可靠在存储大规模的keyed 状态。

不幸的是， 对于不同的配置，RocksDB 的性能表现非常不同。 且关于如何正确进行RocksDB配置调优的文档很少。 比如， 默认是配置是针对SSDs的， 对于机械硬盘不是很合适。



#### 增量的 Checkpoints
相比于全量的检查点， 增量检查点能够显著地减少检查点时间， 可能的代价是更长的恢复时间。 增量检查点的核心思想是只记录前一次检查点后发生的变化，而不是全量的，自包含的状态后端的备份。 增量检查点是基于前一个检查点的。 Flink 以一种随着时间自整合的方式利用 RocksDB 内部的备份机制。 因此， 增量检查点不会无限地增长，量的检查点最终会自动地整合在一起。 

虽然我们强烈建议在大型状态中使用增量检查点，蛤当前这是一个新的功能， 默认情况下没有开启。 如要开启这个功能， 用户可能在实例化 RocksDBStateBackend 时指定： 

```java
   RocksDBStateBackend backend =
        new RocksDBStateBackend(filebackend, true);
```

#### RocksDB Timers

对于RocksDB用户可以选择将定时器保存在堆内存上还是RocksDB中。 基于堆内存的定时器在少量的定时器时性能更好， 将定时器存入RocksDB可以提供更高的扩展性，RocksDB中的定时器数量可能超出可用的主内存（溢出到磁盘）。 

使用RocksDB作为状态后端， 可以通过 Flink 配置项 **state.backend.rocksdb.timer-service.factory** 来选择定时器的存储类型。 可选项为：heap(将定时器存入堆内存，默认), rocksdb(将定时器存入RocksDB)。

增量检查点+基于堆内存的定时器存储的 RocksDB 当前还不支持定时器的异步快照。  其它的状态像键状态还是异步快照的。 这不是前一版本的退化， 将会在 **FLINK-10026** 中解决。


#### 预定义选项

Flink provides some predefined collections of option for RocksDB for different settings, and there existed two ways to pass these predefined options to RocksDB:

Flink 提供一了些针对不同设置的预定义的RocksDB选项集， 有两种方式来传入预定义的选项到RocksDB：
- 通过flink-conf.yaml 的 **state.backend.rocksdb.predefined-options** 来指定。 默认值是 DEFAULT （PredefinedOptions.DEFAULT）
- 通过代码来设置，如 **RocksDBStateBackend.setPredefinedOptions(PredefinedOptions.SPINNING_DISK_OPTIMIZED_HIGH_MEM)**

通过代码指定的选项集会覆盖flink-conf.yaml 的配置。


#### Passing Options Factory to RocksDB

Flink中， 有两种方式将选项工厂传给 RocksDB ： 
- 通过 flink-conf.yaml 的 **state.backend.rocksdb.options-factory** 配置。  默认值是 **org.apache.flink.contrib.streaming.state.DefaultConfigurableOptionsFactory**， 所有的可靠配置定义在 **RocksDBConfigurableOptions** 中。 还可以像下面代码一样自定义配置选项及配置工厂类，并将类名配置到 **state.backend.rocksdb.options-factory**


```java
    public class MyOptionsFactory implements ConfigurableOptionsFactory {

        private static final long DEFAULT_SIZE = 256 * 1024 * 1024;  // 256 MB
        private long blockCacheSize = DEFAULT_SIZE;

        @Override
        public DBOptions createDBOptions(DBOptions currentOptions) {
            return currentOptions.setIncreaseParallelism(4)
                   .setUseFsync(false);
        }

        @Override
        public ColumnFamilyOptions createColumnOptions(ColumnFamilyOptions currentOptions) {
            return currentOptions.setTableFormatConfig(
                new BlockBasedTableConfig()
                    .setBlockCacheSize(blockCacheSize)
                    .setBlockSize(128 * 1024));            // 128 KB
        }

        @Override
        public OptionsFactory configure(Configuration configuration) {
            this.blockCacheSize =
                configuration.getLong("my.custom.rocksdb.block.cache.size", DEFAULT_SIZE);
            return this;
        }
    }
    
```

- 通过代码设置选项工厂，如 **RocksDBStateBackend.setOptions(new MyOptionsFactory());**


代码设置的选项优先于flink-conf.yaml， 选项工厂优先于预定义选项。

RocksDB 是一个本地库， 从进程直接分配内存，而不是从JVM中。 任何指派给 RocksDB  的内存都要考虑到， 通常是将 TaskManager 堆内存大小减小相应的数据。 

您分配给rocksdb的任何内存都必须考虑在内，通常是通过将任务管理器的JVM堆大小减少相同的数量。如果不这样做，可能会导致yarn/meos/etc终止用于分配比配置更多内存的JVM进程。

## 容量规划
下面讨论下如何对Flink的作业进行资源预估，保证作业可靠地运行。 一些基本的经验法则：

- 正常的处理应该有足够的容量以确保不引起持续的背压。 详细可参考[背压监控](https://ci.apache.org/projects/flink/flink-docs-release-1.8/monitoring/back_pressure.html)
- 要在不引起背压的基础上预留一些额外的资源。 额外的资源需要满足在应用故障恢复期间输入数据堆积的需求。 具体需要多少取决于一般的故障恢复时间（故障后TaskManager重新加载状态等）。
    IMPORTANT： 基准是建立在开户检查点的情况上的， 因为检查点也需要占用一部分的资源（网络带宽等）
-  偶尔出现短暂的背压一般没有什么问题， 在负载峰值、追赶进度或外部系统偶现缓慢的时候，这是一个正常现象。 
- 一些特定的算子（如大窗口） 会引起下游算子的峰值： 在窗口构建期间内，下游的算子没有什么需要处理， 当窗口结束触发会出现大的负载。 
规划中需要将下游算子的并行度考虑在内（同时发送多少窗口， 处理窗口峰值的速度）。

IMPORTANT： 为了后续能够增加资源， 请确保将流处理程序的最大并行度设置为一个合理的值。 最大并行度决定了程序的最高可扩展的容量（通过保存点）。

Flink 内部是以最大并行度的粒度来跟踪键组的并行状态。 Flink的设计是尽量让以最大并行度运行时的效率更高，即使程序是以较小的并行度来运行的。

## 压缩
Flink 为所有的查检点和保存点提供了可选的压缩选项（默认关闭）。 当前压缩都是采用的快速压缩算法（1.1.4）， 我们计划在未来支持用户自定义压缩算法。 压缩是以键组为粒度的， 每个键组可以独立的解压， 这对于扩展来说是很重要的一个特性。


通过 ExecutionConfig 来开启压缩选项:
```java
ExecutionConfig executionConfig = new ExecutionConfig();
executionConfig.setUseSnapshotCompression(true);
```

压缩对于增量快照没有影响，因为使用的都是 RocksDB 的快速压缩格式。 

## 任务本地容错

### 动机
对于检查点， 每一个任务都会生成一个状态快照并写入到分布式存储上。 每一个任务会将状态快照在分布式存储上的位置句柄发送给 jobManager。 最后， JobManager收集所任务的状态快照句柄，把它们绑定到查检点对象上。 

当恢复时， JobManager 打开最后一个查检点对象， 并将句柄返回给对应的任务， 任务再从分布式存储上读取回状态快照进行加载。  使用分布式存储有两个好处： 一是分布式存储具有容错能力，二是所有的节点都能访问到并能方便地进行再分布（如扩容）。

但使用远程的分布式存储也有一个缺点： 所有的任务都必须通过网络从远程机器上取回状态。 在许多场景， 恢复时失败的任务会再分配到之前运行的 TaskManager（不是一定的，机器故障时）。 但我们还是去读取远程状态。 状态很大时，这会导致长耗时的恢复时间， 即使只是某一个机器上的小故障。


### 目标

任务本地状态恰好是解决这个恢复耗时问题的， 主要思想是： 对于每一个检查点， 每个任务不仅将任务的状态写入到分布式存储上， 还在任务本地机器上保存一个快照的备份（磁盘或内存）。  需要注意的是，快照的主存储还是分布式存储， 因为本地存储在故障是不保证持久性，且其它节点不能远程访问。 

但是对于恢复时再分配到前一次相同的节点上的任务而言， 可以从本地备份加载状态，不需要远程读取分布式存储上的状态。 大多数的故障都不是节点故障，节点为故障一般也只是影响一个或少数几个节点， 基本上大多数的任务在恢复时还是分配到之前运行的节点上的，这样就就可以在本地读取状态快照。 高效的本地恢复可以显著地减少恢复时间。 

需要注意的是，要玩选择的状态后端和检查点策略，  为每个检查点生创建并存储本地备份会产生一些额外的代价。 比如， 大多数的实现只是简单地远程和本地分别写入一次。


![local_recovery.png](https://ci.apache.org/projects/flink/flink-docs-release-1.8/fig/local_recovery.png)

### 状态快照的主（分布式存储）与从（任务本地）的关系

任务本地状态只能被认为是一个从属的备份， 查检点状态的主存储还是在分布式存储。 
- 对于检查点而言， 主存储的写入必须成功， 写入备份失败不会导致检查点失败。 ，而主存储创建失败时检查点也是失败的，即使备份是成功的。
- 只有主存储是被JobManage确认并管理的。 从属备份属于TaskManage，其生命周期独立于主存储。如可以在主存储上保留最后3个检查点，任务本地状态最保存最后一个查检点。
- 恢复时，如果对应的备份可用，Flink会先尝试从本地加载状态。 如果从本地加载有问题， Flink才会再从主存储上恢复任务状态。 只有备份（可选）、主存储都失败时，恢复才会失败。 这时， 根据配置，Flink会尝试从更前面的检查点进行恢复。
- 任务本地存储可能只有部分不完整的状态（如，写入本地文件时出错）。 这时， Flink会先尝试从本地加载部分状态，本地没有的状态再从主存储上加载。 主存储一定是完整的， 是本地状态的超集。
- 相比于主存储，本地状态可能有各种不同的格式，不一定是二进制的形式。 如可以是内存中的一个堆对象，而不是存放在文件中。 
- 如果一个TaskManage丢失了，其对的所有的任务的本地状态也会丢失。

### 配置任务本地容错
Task-local recovery is deactivated by default and can be activated through Flink’s configuration with the key state.backend.local-recovery as specified in CheckpointingOptions.LOCAL_RECOVERY. The value for this setting can either be true to enable or false (default) to disable local recovery.

 默认情况下任务本地恢复是未开启的。 可以通过将Flink 中**CheckpointingOptions.LOCAL_RECOVERY** 的
**state.backend.local-recovery** 来开启任务本地恢复， 取值为true/false。
```java
    public static final ConfigOption<Boolean> LOCAL_RECOVERY = ConfigOptions
            .key("state.backend.local-recovery")
            .defaultValue(false)
            .withDescription("This option configures local recovery for this state backend. By default, local recovery is " +
                "deactivated. Local recovery currently only covers keyed state backends. Currently, MemoryStateBackend does " +
                "not support local recovery and ignore this option.");
```

### 不同的状态后端下任务容错的细节
限制： 当前，任务本地恢复只覆盖了键状态后端。  当前状态大部分都是键状态。 不久，我们也会覆盖算子状态和定时器。

下面的状态后端可以支持任务本地恢复：
- FsStateBackend：支持键状态。 通过再次写入到本地文件来实现。 会引入额外的写消耗，并占用本地磁盘空间。 将来，我们可能会提供内存版本的实现。
- RocksDBStateBackend： 支持键状态。 对于全量快照，，状态重复写入本地文件，会引入额外的写消耗，并占用本地磁盘空间。 对于增量快照， 本地状态是依赖于RocksDB的查检点机制。 报建主存储拷贝也是使用这个机制，意味着创建备份是不会引入额外的消耗。 在将检查点目录上传后，仅是简单地保留下这个目录而不是删除。 本地备份可以和RocksDB共享文件（夹）（硬连接），所以增量快照也不会占用额外的磁盘空间。 使用硬连意味着RocksDB目录必须与配置的存储本地状态的目录在一块物理磁盘上。 否则硬连接会建立失败（FLINK-10954）。 当前，RocksDB 目录配置到了多个物理磁盘上时，是不能使用本地恢复的。 

### 保留调度的分配结果

任务本地恢复假定在故障时会保留任务的调度分配。 每个任务会记得他之前分配的槽位，恢复重启时会请求分配到桢的槽位。 如果这个槽位不可用了， 任务会向ResourceManager请求分配一个新的槽位。 通过这种方式， 如果 TaskManager 不可用了， 不能恢复到之前槽位的任务不会挤占别的恢复中的任务的槽位。 因为认为只有TaskManager不可用时，上一个槽位才会消失， 在这种情况下，对应槽位的任务无论如何必须去申请一个新的槽位了。 通过这种高度策略， 可以让最大数量的任务有机会恢复到其对应的之前的槽位上， 从而避免了相互窃取之前槽位而引起的级联效应。
