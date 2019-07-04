2019-06-24-Elasticsearch的数据类型（一）概要.md


ES文档中的每个字段都有数据类型的：
- 简易类型： text, keywork, date, long, double, boolean 或 ip
- 一些支持Josn结构的类型： object, nested
- 其它的特殊类型： geo_po9int, geo_shape, 或 completion

# 核心数据类型
字符串
    text 和 keywork
数字类型
    long, interger, short , byte, double, float, half_float, scaled_float
日期类型
    date
布尔类型
    boolean
二进制类型
    binary
范围类型
    interger_range, float_range, long_range, double_range, date_range

# 复杂类型
数组
    数组不需要特别指定一个类型
对象
    object:  单独的Json对象
内嵌
    nested: Json对象人数组

# GEO数据类型
Geo-Point
Geo-shape

# 特殊类型
IP
completion 数据类型
Token count 
mapper-murmur3
Percolator
join 

# Multi-fields
经常会根据不同的目的将同一个字段索以不同的方式来索引。 如 一个string字段索引为一个text用于全文搜索，也可以索引为keywork用于排序和聚合。 


