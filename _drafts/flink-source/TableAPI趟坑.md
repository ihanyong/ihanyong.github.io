TableAPI趟坑


- 使用 window group，要注意在Stream的watermark
- window group 时， 注意在 底层流上指定时间角色（事件时间，处理时间）， 并声明时间属性
- 时间属性的声名方式： 1. 流转表时的声名方式（.rowtime/.protime）; 2. tablesource的方式通过 DefinedProctimeAttribute、DefinedRowtimeAttributes 接口实现
- DefinedProctimeAttribute、DefinedRowtimeAttributes，DefinedProctimeAttribute、DefinedRowtimeAttributes 现在只支持指定一个时间属性，属性名需要在TableSchema里面有，而且类型为sql的timestamp。（如果原时间属性不是TimeStamp, 可以通过一个附加一个TimeStamp的点位属性，真实值还是通过TimestampExtractor提取的方式来绕过。）


## 2019年3月20日
- 双流join， 注意水位与时间戳不要搞混在一起。 是两个概念，分别去理解的。 
    + 时间戳是和事件数据一起的
    + 水位是单独的一个StreamElement
    + StreamElement 的几个类型： StreamRecord; Watermark; StreamStatus; LatencyMarker;
    + 使用event time 的时候一定注意 watermark 的问题， 容易造成流假死~
    + 





[流计算技术实战 - 超大维表问题](https://www.cnblogs.com/fxjwind/p/7771338.html)


https://blog.acolyer.org/2015/05/26/pregel-a-system-for-large-scale-graph-processing/

http://giraph.apache.org/#


jobs that process so much data and run for such a long time that they are likely to experience at least one task failure along the way. In that case, rerunning the entire job due to a single task failure would be wasteful. Even if recovery at the granularity of an individual task intorduces overheads that make fault-free processing slowe, it can still be a reaonable trade-off if the rate of task failures is high enough.



However, there are also proplems with the MapReduce execution model itself, which are not fixed by adding another level of abstraction and which manifest themselves as poor performance for some kinds of processing. On the one hand, MapReduce is very robust: you can use it to process almost arbitrarily large quantities of data on an unreliable multi-tenant system with frequent task terminations, and it weill still get the job done (albeit slowly). On the other hand, other tools are sometimes orders of magnitude faster for some kinds of processing.

At the point when the event is generated, it becomes a fact. Event if the customer later decides to chage or cancel the reservation, the fact remains true that they formerly held a reservation for a particular seat, and the change or cancellation is separate event that is added later.

A consumer of the event stream is not allowed to reject an event: by the time the consumer sees the event, it is already an immutable part of the log, and it may have already been seen by other consumers. Thus, any validation of a command needs to happen synchronously, before it becomes an event -- for example, by using a serializable transaction that atomically validates the command and publishes the event.

Althernatively, the user request to reserve a seat could be split into tow event: first a tentative reservation, and then a separate confirmation event once the reservation has been validated. This split allows the validation to take place in an asynchronous process.


State, Streams, Immutability

一般认为数据库是用来存储应用当前的状态的， 可以用来方便查询。 状态是可变是，因此数据库支持插入更新删除。 

这个可变的状态是一系列的不可变的事件的结果。

transaction logs record all the changes made to the database. High-speed appends are the only way to change the log. From this perspective, the contents of the database hold a caching of the latest recored values in the logs. The truth is the log . The database is a cache of a subset of the log. That cached subset happens to be the latest value of each record and index value from the log.



Having an explicit translation step from an event log to a dateabase makes it easier to evolve your applicaton over time: if you want to introduce a new feature that presents your existing data in some new way, you can use the vent log to build a separate read-optimized view for the new feature, and run it alongside the existing systems without having to modify them. Running old and new systms side by side is often easier than performing a complicated schema migration in an existing system. Once the old system is no longer needed, you can simply shut it down and reclaim its resources.

Storing data is normally quite straightforward if you dont's have to worry about how it is going to be queried and accessed; many of the complexities of schema design, indexing, and storage engines are the result of wanting to support cetain query and access patterns. Fro this reason, you gain a lot of flexibility by separating the form in which data is written from the form it is read, and by allowing several different read views. This idea is sometimes known as __command query responsibility segregation__(CQRS).

processing streams to produce other, derived streams. A piece of code of that processes streams like this is known as an operator ro a job. It is closely related to the Unix processes and MapReduce jobs we discussed in 






tweets are written as they are sent, so that reading the timeline is a single lookup. Materializing and maintaining this cache requires the following event processing:

