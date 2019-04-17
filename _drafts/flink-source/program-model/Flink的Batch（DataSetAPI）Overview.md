Flink的Batch（DataSetAPI）Overview

Flink中的DataSet 程序是用来实现 Data set 的转换的（filtering, mapping, joing , grouping 等）。 Data Set 是从某个源产生的有限数据集全（读取文件， 本地集合）。结果通过sink返回， 如将结果数据写入（分布式）文件系统， 输出。  Flink程序可以在各种上下文中运行： 独立模式， 嵌入其它程序等。 可以在本地JVM上执行，也可以在多机器的集群上执行。

# 示例程序

下面是一个完整的可执行的单词数统计程序。 可以拷贝到本地运行（相应的包要引入）。

```java

public class WordCountExample {
    public static void main(String[] args) throws Exception {
        final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

        DataSet<String> text = env.fromElements(
            "Who's there?",
            "I think I hear them. Stand, ho! Who's there?");

        DataSet<Tuple2<String, Integer>> wordCounts = text
            .flatMap(new LineSplitter())
            .groupBy(0)
            .sum(1);

        wordCounts.print();
    }

    public static class LineSplitter implements FlatMapFunction<String, Tuple2<String, Integer>> {
        @Override
        public void flatMap(String line, Collector<Tuple2<String, Integer>> out) {
            for (String word : line.split(" ")) {
                out.collect(new Tuple2<String, Integer>(word, 1));
            }
        }
    }
}


```

# DataSet 的算子

调用DataSet的算子操作会从一个或多个DataSet产生一个新的DataSet。  可能通过程序来精细的编排多个算子。

下面是简要的说明

## Map
输入一个元素并返回一个元素

## FlatMap
输入一个元素，返回若干个（0，1。。。。）

## MapPartition
Transforms a parallel partition in a single function call. The function gets the partition as an Iterable stream and can produce an arbitrary number of result values. The number of elements in each partition depends on the degree-of-parallelism and previous operations.

## Filter
为每一个元素应用谓词函数，为true保留，false过滤。
应用谓词函数时不就改变数据的值 ， 否则会导致错误的结果。

## Reduce
通过不重复调地将两个元素结合为一个，将一组元素归约为一个元素。 
reduce 可以应用在整个数据集上也可以是分组应用。

## ReduceGroup
将一组元素归约为一个元素，可以应用在整个数据集上也可以是分组应用。

## Aggregate
将一组值聚合为一个值。 可以看做是内置的 reduce 函数。 可以应用在整个数据集上也可以是分组应用。

## Distinct
对数据进行去重。 可以基于全部或部分元素字段对全部数据集进行去重。

Distinct 是用reduce 来实现的。 通过setCombineHint可以指定运行时的执行方式。 基于 Hash的策略一般比较快， 尤其是在区分度较的情况下。

## Join
将两个数据集合中key值相等的元素相互结合进行Join。 可使用JoinFunction来将一对元素转换为一个元素， 或FlatJoinFunction 将一对元素转换为多个元素。

可以通过JoinHint来指定运行时的执行方式。  JoinHint 指示是通过分片还是广播来进行Join，是使用排序算法还是Hash算法。 如果没有指定， 系统会自动评估输入的规模并选择最优策略。

Join 算了只支持 等值Join， 其它的Join类型需要通过 OuterJoin 和COGroup来表述。

## OuterJoin
leftOuterJion, fullOuterJoin, rightOuterJoin
类似于 一般的连接，基于key的相等来对两边的元素进行组对。 如果没有相匹配的key, outer端的元素会保留，并与null进行组对。 可使用JoinFunction来将一对元素转换为一个元素， 或FlatJoinFunction 将一对元素转换为多个元素。


## CoCgroup
reduce 的一个变种。 将两边的输入进行分组，然后将分组进行连接。 UDF 应用在分组数据的对子上。

## Cross
返回输入两边的笛卡尔积。 可以通过CrossFunction  将一对元素转换为一个。
Note： Cross 可能会是一个超级计算密集的操作，即使对于一个大规模的云计算平台来说也是一个挑战。 建议通过 crossWithTiny() and crossWithHuge() 来提示系统 输入数据 的规模。

## Union
Produces the union of two data sets.

## Rebalance
再均衡并行分片以消除数据倾斜。 后面只能接类Map的算子。

## Hash-Partition
通过Hash对数据进行分片， keys 可以通过 位置、表达式和select function来指定

## Range-Partition
通过范围对数据进行分片， keys 可以通过 位置、表达式和select function来指定

## Custom-Partition
用户自定义分片策略

## Sort Partition
在每一个分片内部进行排序列。 可链式调用实现多字段排序。

## First-n
返回前n个数据项。

# Data Sources

Data sources 创建了初始数据集， 如从文件或Java集合创建。 创建数据集的一般机制抽象于 InputFormat。 Flink内置了几个 formats 从觉的文件格式创建数据集， 在ExecutionEnvironment有一些快捷方法。

基于文件：
- readTextFile(path) / TextInputFormat
- readTextFileWithValue(path) / TextValueInputFormat
- readCsvFile(path) / CsvInputFormat
- readFileOfPrimitives(path, Class) / PrimitiveInputFormat
- readFileOfPrimitives(path, delimiter, Class) / PrimitiveInputFormat
- readSequenceFile(Key, Value, path) / SequenceFileInputFormat

基于集合：
- formCollection(Collection)
- fromCollection(Iterator, Class)
- fromElements(T ...)
- fromParallelCollection(SplittableIterator, Class)

一般：
- readFile(inputFormat, path)
- createInput(inputFormat)

示例：
```java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// read text file from local files system
DataSet<String> localLines = env.readTextFile("file:///path/to/my/textfile");

// read text file from a HDFS running at nnHost:nnPort
DataSet<String> hdfsLines = env.readTextFile("hdfs://nnHost:nnPort/path/to/my/textfile");

// read a CSV file with three fields
DataSet<Tuple3<Integer, String, Double>> csvInput = env.readCsvFile("hdfs:///the/CSV/file")
                           .types(Integer.class, String.class, Double.class);

// read a CSV file with five fields, taking only two of them
DataSet<Tuple2<String, Double>> csvInput = env.readCsvFile("hdfs:///the/CSV/file")
                               .includeFields("10010")  // take the first and the fourth field
                           .types(String.class, Double.class);

// read a CSV file with three fields into a POJO (Person.class) with corresponding fields
DataSet<Person>> csvInput = env.readCsvFile("hdfs:///the/CSV/file")
                         .pojoType(Person.class, "name", "age", "zipcode");

// read a file from the specified path of type SequenceFileInputFormat
DataSet<Tuple2<IntWritable, Text>> tuples =
 env.readSequenceFile(IntWritable.class, Text.class, "hdfs://nnHost:nnPort/path/to/file");

// creates a set from some given elements
DataSet<String> value = env.fromElements("Foo", "bar", "foobar", "fubar");

// generate a number sequence
DataSet<Long> numbers = env.generateSequence(1, 10000000);

// Read data from a relational database using the JDBC input format
DataSet<Tuple2<String, Integer> dbData =
    env.createInput(
      JDBCInputFormat.buildJDBCInputFormat()
                     .setDrivername("org.apache.derby.jdbc.EmbeddedDriver")
                     .setDBUrl("jdbc:derby:memory:persons")
                     .setQuery("select name, age from persons")
                     .setRowTypeInfo(new RowTypeInfo(BasicTypeInfo.STRING_TYPE_INFO, BasicTypeInfo.INT_TYPE_INFO))
                     .finish()
    );

// Note: Flink's program compiler needs to infer the data types of the data items which are returned
// by an InputFormat. If this information cannot be automatically inferred, it is necessary to
// manually provide the type information as shown in the examples above.
```
#### Configuring CSV Parsing
关于CSV的转换有一些可配置项：
- types(Class ... types) 
- lineDelimiter(String del)
- fieldDelimiter(String del)
- includeFields(boolean ... flag), includeFields(String mask), includeFields(long bitMask)
- paraseQoutedStrings(char qouteChar) 
- ignoreComments(String commentPrefix)
- ignoreInvalidLines() 
- ignoreFirstLin() 

#### 递归遍历文件夹
对于基于文件的输入，当输入的路径是文件夹时，默认情况下不会枚举子嵌套文件夹的文件。相反，只读取当前文件夹中的文件，而忽略嵌套文件夹的文件。嵌套文件的递归枚举可以通过recursive.file.enumeration配置参数启用，如下例所示
```java
// enable recursive enumeration of nested input files
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// create a configuration object
Configuration parameters = new Configuration();

// set the recursive enumeration parameter
parameters.setBoolean("recursive.file.enumeration", true);

// pass the configuration to the data source
DataSet<String> logs = env.readTextFile("file:///path/with.nested/files")
              .withParameters(parameters);
```

# 读取压缩文件
目前，如果文件使用了对应的扩展名， Flink支持对输入文件进行自动解压。这意味着不需要进一步配置输入格式，任何fileinputformat都支持压缩，包括自定义输入格式。请注意，压缩文件可能无法并行读取，从而影响作业的可伸缩性。

| Compression<br/>method | File extensions | Parallelizable |
|---|---|---|
| DEFLATE | .deflate | no |
| GZip | .gz, .gzip | no |
| Bzip2 | .bz2 | no |
| XZ | .xz | no |



# Data Sinks
Data sinks 消费 DataSets 并用来存储或返回数据。 Data sink 操作使用 OutputFormat来描述。 Flink 内置许多 output format 封装于 DataSet的算子中。

- writeAsText() / TextOutputFormat
- writeAsFormattedText() / TextOutputFormat
- writeAsCsv() / CsvOutputFormat
- print()/ printToErr() / print(String msg) / printToErr(String msg)
- write() / FileOutputFormat
- output() / OutputFormat


数据集可以输入到多个操作中。程序可以在写入或打印数据集的同时对其运行其他转换。 

Exaples:
```java
///////////////////// Standard data sink methods:

// text data
DataSet<String> textData = // [...]

// write DataSet to a file on the local file system
textData.writeAsText("file:///my/result/on/localFS");

// write DataSet to a file on a HDFS with a namenode running at nnHost:nnPort
textData.writeAsText("hdfs://nnHost:nnPort/my/result/on/localFS");

// write DataSet to a file and overwrite the file if it exists
textData.writeAsText("file:///my/result/on/localFS", WriteMode.OVERWRITE);

// tuples as lines with pipe as the separator "a|b|c"
DataSet<Tuple3<String, Integer, Double>> values = // [...]
values.writeAsCsv("file:///path/to/the/result/file", "\n", "|");

// this writes tuples in the text formatting "(a, b, c)", rather than as CSV lines
values.writeAsText("file:///path/to/the/result/file");

// this writes values as strings using a user-defined TextFormatter object
values.writeAsFormattedText("file:///path/to/the/result/file",
    new TextFormatter<Tuple2<Integer, Integer>>() {
        public String format (Tuple2<Integer, Integer> value) {
            return value.f1 + " - " + value.f0;
        }
    });




///////////////////// Using a custom output format
DataSet<Tuple3<String, Integer, Double>> myResult = [...]

// write Tuple DataSet to a relational database
myResult.output(
    // build and configure OutputFormat
    JDBCOutputFormat.buildJDBCOutputFormat()
                    .setDrivername("org.apache.derby.jdbc.EmbeddedDriver")
                    .setDBUrl("jdbc:derby:memory:persons")
                    .setQuery("insert into persons (name, age, height) values (?,?,?)")
                    .finish()
    );

```

### output的本地排序 
Data sink 的输出可以在指定的字段（tuple field position, field expressions）上以指定的顺序进行本地（分片）排序。 所有的 output format 都是支持的。

暂不支持全局排序 


```java
DataSet<Tuple3<Integer, String, Double>> tData = // [...]
DataSet<Tuple2<BookPojo, Double>> pData = // [...]
DataSet<String> sData = // [...]

// sort output on String field in ascending order
tData.sortPartition(1, Order.ASCENDING).print();

// sort output on Double field in descending and Integer field in ascending order
tData.sortPartition(2, Order.DESCENDING).sortPartition(0, Order.ASCENDING).print();

// sort output on the "author" field of nested BookPojo in descending order
pData.sortPartition("f0.author", Order.DESCENDING).writeAsText(...);

// sort output on the full tuple in ascending order
tData.sortPartition("*", Order.ASCENDING).writeAsCsv(...);

// sort atomic type (String) output in descending order
sData.sortPartition("*", Order.DESCENDING).writeAsText(...);
```



# Iteration Operators
迭代实现了Flink程序的循环。 迭代算子封装了程序的一部分并重复地执行，将一次迭代的结果 反馈给下一次迭代。 Flink中有两种类型的迭代： BulkIteration 和 DeltaIteration。

## Bulk Iterations
在开始迭代的DataSet上调用 iterate(int)方法来创建 BulkIteration。 在返回的 IterativeDataSet上可以调用常规的转换算子。 iterate 的 int 参数用来指定最大的迭代次数。

调用 IterativeDataSet.closeWith(DAtaSet) 来指定迭代的结束 ，进入下一轮迭代。
可以通过closeWith(DataSet, DataSet)指定一个结束条件，通过计算第二个DataSet为空时结束迭代。如果没有指定结束条件，则迭代指定的最大次数后结束迭代。

下面是一个通过迭代来估算Pi的示例：

```java
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// Create initial IterativeDataSet
IterativeDataSet<Integer> initial = env.fromElements(0).iterate(10000);

DataSet<Integer> iteration = initial.map(new MapFunction<Integer, Integer>() {
    @Override
    public Integer map(Integer i) throws Exception {
        double x = Math.random();
        double y = Math.random();

        return i + ((x * x + y * y < 1) ? 1 : 0);
    }
});

// Iteratively transform the IterativeDataSet
DataSet<Integer> count = initial.closeWith(iteration);

count.map(new MapFunction<Integer, Double>() {
    @Override
    public Double map(Integer count) throws Exception {
        return count / (double) 10000 * 4;
    }
}).print();

env.execute("Iterative Pi Example");

```

## Delta Iterations
增量迭代利用了这样一个事实，即某些算法不会在每次迭代中更改解决方案的每个数据点。

除了在每次迭代中反馈的部分解决方案（称为工作集）之外，增量迭代在迭代之间保持状态（称为解决方案集），可以通过增量更新。迭代计算的结果是最后一次迭代后的状态。

定义DeltaIteration类似于定义BulkIteration。对于增量迭代，两个数据集构成每个迭代（工作集和解决方案集）的输入，并且在每个迭代中生成两个数据集作为结果（新工作集、解决方案集增量）。

要创建DeltaIteration，调用Iteratedelta(DataSet、int、int)或Iteratedelta(dataset、int、int[])。在初始解集上调用此方法。参数是初始增量集、最大迭代次数和关键位置。返回的DeltaIteration对象使您能够通过iteration.getWorkset（）和iteration.getSolutionset（）方法访问表示工作集和解决方案集的数据集。
```java
// read the initial data sets
DataSet<Tuple2<Long, Double>> initialSolutionSet = // [...]

DataSet<Tuple2<Long, Double>> initialDeltaSet = // [...]

int maxIterations = 100;
int keyPosition = 0;

DeltaIteration<Tuple2<Long, Double>, Tuple2<Long, Double>> iteration = initialSolutionSet
    .iterateDelta(initialDeltaSet, maxIterations, keyPosition);

DataSet<Tuple2<Long, Double>> candidateUpdates = iteration.getWorkset()
    .groupBy(1)
    .reduceGroup(new ComputeCandidateChanges());

DataSet<Tuple2<Long, Double>> deltas = candidateUpdates
    .join(iteration.getSolutionSet())
    .where(0)
    .equalTo(0)
    .with(new CompareChangesToCurrent());

DataSet<Tuple2<Long, Double>> nextWorkset = deltas
    .filter(new FilterByThreshold());

iteration.closeWith(deltas, nextWorkset)
    .writeAsCsv(outputPath);
```

# Operating on Data Objects in functions

Flink 运行时与UDF以Java对象的形式来交换数据。 UDF以方法参数接收输入对象，通过结果返回输出对象。 因为这些对象同时被UDF和运行时的代码访问，所以理解并遵循UDF访问这些对象的规则是非常重要的。

UDF以常规方法参数（MapFunction）或通过Iterable通过（GroupReduceFunction）从Flink运行时接收对象。我们将运行时传递给UDF的对象称为输入对象。UDF可以通过返回值（MapFunction）或Collector（FlatMapFunction）发送对象给Flink运行时。我们将UDF发送给运行时的对象称为输出对象。

Flink’s DataSet API 提供两种模式，在Flink运行时创建或重用输入对象上不同。这个行为影响了的保证和约束：UDF与输入输出对象进行交互方式。

### 禁用对象重用（默认）
Flink默认是禁用对象重用的。 这种模式保证了UDF被调用时接收到的总是新的输入对象。禁用对象重用模式提供了更好的保证，使用起来更安全。但是它有一定的处理开销，并且会引起更多的垃圾收集处理。 

| 操作 | 保证与约束 |
| --- | --- |
| 读取输入对象 | 在方法调用内部，可以保证对象是值是不变的，包括通过Iterable提供的对象。 如将Iterable的内容收集到List呀Map中是安全的。 注意方法调用结束后，对象可能会被修改。 跨方法保存的对象是不安全的。 |
| 修改输入对象 | 可以修改输入对象 |
| 发送输入对象 | 可以发送输入对象。 发送后输入对象的值可能会被修改。 改善之后再读取输入对象是不安全的 |
| 读取输出对象 | 通过收集器或返回值发送的对象的值可能已改变。读取输出对象是不安全的。  | 
| 修改输出对象 | 对象被发送后，可以修改后再发送 |

禁用对象重用模式的编码指南:
- 不要保留和跨方法读取输入对象。
- 发送数据后不要再读取了对象了。

### 启用对象重用
在启用对象重用模式下， Flink运行时会减少实例化对象的数量。这样可以提升性能并减小GC压力。ExecutionConfig.enableObjectReuse()来开户对象重用模式。

| 操作 | 保证与约束 |
| --- | --- |
| 读取方法参数的输入对象 | 作为方法参数接收的输入对象在方法调用中是不会被 修改的。方法调用结束后，对象可能会被修改。 跨方法保存的对象是不安全的。 |
| 读取Iterable的输入对象 | 从Iterable接收的输入对象只有在next()调用前是有效的。 Iterable可能会多次输出同一个对象。 |
| 修改输入对象 | 绝对不能修改输入对象，除了MapFunction, FlatMapFunction, MapPartitionFunction, GroupReduceFunction, GroupCombineFunction, CoGroupFunction, and InputFormat.next(reuse).  |
| 发送输入对象 | 绝对不能发送输入对象，除了MapFunction, FlatMapFunction, MapPartitionFunction, GroupReduceFunction, GroupCombineFunction, CoGroupFunction, and InputFormat.next(reuse).  |
| 读取输出对象 | 通过收集器或返回值发送的对象的值可能已改变。读取输出对象是不安全的。  | 
| 修改输出对象 | 对象被发送后，可以修改后再发送 |

启用对象重用模式的编码指南:
- 不要保留从Iterable接收的输入对象
- 不要保留并跨方法读取输入对象
- 不要修改或发送输入对象，除了 MapFunction, FlatMapFunction, MapPartitionFunction, GroupReduceFunction, GroupCombineFunction, CoGroupFunction, and InputFormat.next(reuse)
- 为了减少对象的实例化，可以一直重复地修改并发送一个专用的输出对象，但不能读取它。

# Debugging
在分布式集群上大数据集上执行程序之前，最好是能确保算法实现的正确性。 实现算法分析程序 通常是一个演进的过程（检查结果，调试，优化）。

Flink 提供了一些友好的功能支持IDE中本地调用，注入测试数据，收集结果数据 以简化数据分析程序的开发过程。

### 本地执行环境
LocalEnvironment 在本地JVM中启动Flink系统。 在IDE中启动LocalEnvironment后，可以打断点来调试代码。

```java
final ExecutionEnvironment env = ExecutionEnvironment.createLocalEnvironment();

DataSet<String> lines = env.readTextFile(pathToTextFile);
// build your program

env.execute();

```
### 集合数据源与sink
通过创建输入文件和读取输出文件来为分析程序提供输入并检查其输出是很麻烦的。FLink具有由Java集合支持的特殊数据源和接收器，以便于测试。一旦测试好了一个程序，源和接收器就可以很容易地被从外部数据存储（如HDFS）读取/写入的源和接收器替换。

```java
final ExecutionEnvironment env = ExecutionEnvironment.createLocalEnvironment();

// Create a DataSet from a list of elements
DataSet<Integer> myInts = env.fromElements(1, 2, 3, 4, 5);

// Create a DataSet from any Java collection
List<Tuple2<String, Integer>> data = ...
DataSet<Tuple2<String, Integer>> myTuples = env.fromCollection(data);

// Create a DataSet from an Iterator
Iterator<Long> longIt = ...
DataSet<Long> myLongs = env.fromCollection(longIt, Long.class);
```

```java
DataSet<Tuple2<String, Integer>> myResult = ...

List<Tuple2<String, Integer>> outData = new ArrayList<Tuple2<String, Integer>>();
myResult.output(new LocalCollectionOutputFormat(outData));
```

# 语义注解

语义注解可以用来给Flink提示函数的行为。 告诉系统对于输入对象中， 哪些字段是函数要读取使用的，哪些字段是不修改直接转发到输出的。 语义注解一个加速作为执行有力工具， 因为他可以让系统识别出可跨多个算子重用的排序与分片。使用语义注解最终会规避不必要的shuffling或排序， 显著地提升性能。

语义注解的使用是可选的。 一定要正确的使用， 错误的用法会导致错误的结果，宁可不用也不要用错。

### Forwarded Fields Annotation
转发字段注解用于声明输入字段中，哪些字段不需要修改而直接转发到输出对象的相同或不同位置。 优化器使用这些信息可以推断数据的排序或分片是不是可以保留不变。 

【TODO】
### Non-Forwarded Fields
【TODO】
### Read Fields
【TODO】

# Broadcast Variables  广播变量
广播变量允许一个算子所有的并行实例都可以访问一个额外的数据集。 对于辅助数据集或依赖数据集来说，很有用。 在算子函数中会以Collection的形式来访问这个数据集。
- 广播： 使用 withBroadcastSet(DataSet, String)来注册一个数据集
- 访问： 在算子里使用getRuntimeContext().getBroadcastVariable(String) 获取数据集

```java
// 1. The DataSet to be broadcast
DataSet<Integer> toBroadcast = env.fromElements(1, 2, 3);

DataSet<String> data = env.fromElements("a", "b");

data.map(new RichMapFunction<String, String>() {
    @Override
    public void open(Configuration parameters) throws Exception {
      // 3. Access the broadcast DataSet as a Collection
      Collection<Integer> broadcastSet = getRuntimeContext().getBroadcastVariable("broadcastSetName");
    }


    @Override
    public String map(String value) throws Exception {
        ...
    }
}).withBroadcastSet(toBroadcast, "broadcastSetName"); // 2. Broadcast the DataSet
```

确保注册和获取数据集时使用的名称是一致的。
广播变量是存放在每个节点的内存中的，所以这个数据集不能太大。 
对于标量值这样简单的事情，可以简单地将参数作为函数闭包的一部分，或者使用withParameters(...)方法传入配置。

# Distributed Cache 分布式缓存
类似于Hadoop， Flink提供了分布式缓存的功能， 使得UDF的每一个并行实例可以以本地化的方式来访问文件。 这个功能可以用于共享那些包含了静态数据（字典，机器学习回归模型）的文件。

程序在ExecutionEnvironment中注册一个本地或远程的文件/目录作为缓存文件。 当程序执行时， Flink会自动地将该文件/目录拷贝到所有工作节点的本地文件系统中。 UDF可以通过注册名从本地文件系统访问该文件/目录。

通过ExecutionEnvironment注册
```java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// register a file from HDFS
env.registerCachedFile("hdfs:///path/to/your/file", "hdfsFile")

// register a local executable file (script, executable, ...)
env.registerCachedFile("file:///path/to/exec/file", "localExecFile", true)

// define your program and execute
...
DataSet<String> input = ...
DataSet<Integer> result = input.map(new MyMapper());
...
env.execute();
```

访问时UDF要是RichFunction
```java
// extend a RichFunction to have access to the RuntimeContext
public final class MyMapper extends RichMapFunction<String, Integer> {

    @Override
    public void open(Configuration config) {

      // access cached file via RuntimeContext and DistributedCache
      File myFile = getRuntimeContext().getDistributedCache().getFile("hdfsFile");
      // read the file (or navigate the directory)
      ...
    }

    @Override
    public Integer map(String value) throws Exception {
      // use content of cached file
      ...
    }
}
```


# Passing Parameters to Functions
可以通过构造函数或withParameters(Configuration)方法向UDF传递参数。 这些参数被作为UDF的一部分序列化并分发到任务所有的并发实例上。

[最佳实践-如何传递命令行参数到UDF](https://ci.apache.org/projects/flink/flink-docs-release-1.7/dev/best_practices.html#parsing-command-line-arguments-and-passing-them-around-in-your-flink-application)

#### 通过构造函数
```java
DataSet<Integer> toFilter = env.fromElements(1, 2, 3);

toFilter.filter(new MyFilter(2));

private static class MyFilter implements FilterFunction<Integer> {

  private final int limit;

  public MyFilter(int limit) {
    this.limit = limit;
  }

  @Override
  public boolean filter(Integer value) throws Exception {
    return value > limit;
  }
}


```

#### 通过 withParameters(Configuration)
本方法接收一个 Configuration对象作为入参， 被传递到 rich function 的open()方法。

```java
DataSet<Integer> toFilter = env.fromElements(1, 2, 3);

Configuration config = new Configuration();
config.setInteger("limit", 2);

toFilter.filter(new RichFilterFunction<Integer>() {
    private int limit;

    @Override
    public void open(Configuration parameters) throws Exception {
      limit = parameters.getInteger("limit", 0);
    }

    @Override
    public boolean filter(Integer value) throws Exception {
      return value > limit;
    }
}).withParameters(config)
```

#### 通过ExecutionConfig传递全局参数

```java


Configuration conf = new Configuration();
conf.setString("mykey","myvalue");
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.getConfig().setGlobalJobParameters(conf);


```


```java
public static final class Tokenizer extends RichFlatMapFunction<String, Tuple2<String, Integer>> {

    private String mykey;
    @Override
    public void open(Configuration parameters) throws Exception {
      super.open(parameters);
      ExecutionConfig.GlobalJobParameters globalParams = getRuntimeContext().getExecutionConfig().getGlobalJobParameters();
      Configuration globConf = (Configuration) globalParams;
      mykey = globConf.getString("mykey", null);
    }
    // ... more here ...
```