---
layout: post
title:  "【Java基础知识梳理】JavaIO"
date:   2019-04-04 22:30:00 +0800
tags:
        - java IO
---

# java IO概念与历史
B: blocking
I: input
O: output

java IO是对数据流入流出的一种抽象表表达方式。 在Java1.4之前，只有阻塞式(B)的IO流API，即通过BIO读取写入数据时，线程会阻塞直到操作完成。 BIO按流的类型可以分为 byte 流和 character 流。 byte流对应的 inputStream/outputStream， 是在Java1.0中引入的； character 流对应的 reader/writer，是在Java1.1引入的。

# 主要的类
```
    byte stream
        InputStream
            FileInputStream
            ByteArrayInputStream
            SequenceInputStream
            StringBufferInputStream @Deprecated
            FilterInputStream
                LineNumberInputStream @Deprecated
                BufferedInputStream
                PushbackInputStream
                DataInputStream
            ObjectInputStream
            PipedInputStream
        OutputStream
            FileOutputStream
            ByteArrayOutputStream
            FilterOutputStream
                PrintStream
                DataOutputStream
                BufferedOutputStream
            ObjectOutputStream
            PipedOutputStream
    character stream
        Reader
            InputStreamReader
                FileReader
            BufferedReader
                LineNumberReader
            CharArrayReader
            PipedReader
            StringReader
            FilterReader
                PushbackReader
        Writer
            CharArrayWriter
            PrintWriter
            FilterWriter
            BufferedWriter
            StringWriter
            PipedWriter
            OutputStreamWriter
                FileWriter
```



# 各种类型流的解析与使用
## byte 流
### InputStream/OutStream 的接口定义


### 文件流(FileInputStream/FileOutputStream)
文件流用于从文件中获取原始的bytes输入输出流，如图片等类型的文件。如果需要读取或输出字符流，使用 FileReader/FileWrite 更合适。 
文件流的实例可能通过构造函数来new,也可以使用 *java.nio.file.Files.newInputStream(path)* *java.nio.file.Files.newOutputStream(path)* (Java1.7)来获取。
文件流的底层实现是通过调用的native方法实现的， 透过源码可以知道所有的read、skip、write、flush、open、close最终都是调用native方法
不支持mark。

Java1.4之后， 文件流提供了getChannel方法从流获取一个NIO的chanenl。


### byte数组流(ByteArrayInputStream/ByteArrayOutputStream)
ByteArrayInputStream/ByteArrayOutputStream 内部实现都是使用byte数组作为底层数据载体。

ByteArrayInputStream， 实例化时传入一个byte数组作为流数据的源，内部会维护一个计数器来记录下一次读取的byte位置。 支持 mark。 close方法对ByteArrayInputStream来说没有什么效果，即调用close后，ByteArrayInputStream的read等方法依然可以调用，不会抛出 IOException。

ByteArrayOutputStream， 实例化时可以指定内部byte数组的大小，默认是32。 内部数组在容量不足时会自动扩容，每次至少是两倍，最大扩容到Integer.MAX_VALUE。 每次扩容都是new一个新的byte数组并将旧数组的数据拷贝到新数组。

ByteArrayOutputStream 提供有了 writeTo(OutputStream out) 方法来将被写入的数据转写到 out流中， 也可以调用 toByteArray 来获取已经写入ByteArrayOutputStream中的byte数据的拷贝。

byte数组流的 public 方法 基本上都是 synchronized的。


### 序列化流(ObjectInputStream/ObjectOutputStream)
序列化流是用来序列化对象的。出于性能，多语言交互等方面的考虑，一般在对象序列化反序列化上很少使用Java原生的序列化机制。

现在市面上常用的序列化方式：
- xml
- json
- [protobuf](https://github.com/protocolbuffers/protobuf)
- [thrift](http://thrift.apache.org/)
- [avro](https://avro.apache.org/)
对于各种序列化方式的优势与缺点这里不做讨论。

可参考：
- [Java Object Serialization Specification ](https://docs.oracle.com/javase/8/docs/platform/serialization/spec/serialTOC.html)
- [Serializable](https://docs.oracle.com/javase/8/docs/api/java/io/Serializable.html)

### 管道流(PipedInputStream/PipedOutputStream)
管道输入流与输出流相互连接在的一起使用，输入流读取的数据就是输出流写入的数据。 一般管道输入流与输出流需要放在两个线程中来实现线程间的通信， 一个线程向output端写入数据，一个线程从input端读取数据。 若输入输出两端没有连接、 另外一端close 或 另外一端的线程结束，则本端在read/write时会抛出IOException。


### 顺序流(SequenceInputStream)
顺序流只有输入流，是有来将若干个输入流按顺序连接起来的逻辑输入流。 顺序流上的read是按顺序调用从第一个底层流开始调用底层流的read方法，直接最后一个底层流调用完毕，返回-1。


### 过滤器流(FilterInputStream/FileOutputStream)
可以理解为一个装饰器模式的应用。  过滤器流内部封装了一个底层流，FilterInputStream/FileOutputStream本身就是简单的重写了InputStream/OutputStream所有的方法，使用所有的参数直接调用底层流。 过滤器的子类可以根据需要重写某些方法来添加额外的功能。

#### 过滤器流的子类
##### 缓冲流
##### 数据流
##### 回推流
##### 打印流 (PrintStream)
hello world 中的 System.out 即是一个 PrintStream。

PrintStream 为其它的输出流添加额外的功能： 方便地打印出种类数据值的表示。
与其它的输出流不同，PrintStream不会抛出 IOException， 有异常时会设置一个可能通过checkError方法访问的内部状态。PrintStream创建的时候可以设置为自动flush， 这样当写入byte数组、调用println方法 或是写入了'\n'时， flush()方法会自动调用。
所有PrintStream打印的字符都是使用平台默认的字符编码bytes转换而来的。 在需要写入字符而不是字节的场景，应该使用PrintWriter。 

# 字符流 [todo]