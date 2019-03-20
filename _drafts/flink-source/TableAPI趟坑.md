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


