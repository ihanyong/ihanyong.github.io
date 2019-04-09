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

# Data Sinks



# Iteration Operators
# Operating on Dta Objects in functions
# Debugging
# Semantic Annotations
# Broadcast Variables
# Distributed Cache
# Passing Parameters to Functions





