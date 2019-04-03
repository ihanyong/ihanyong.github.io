TableAPI-连接外部系统


Flink’s Table API & SQL 程序（batch&Streaming表）可以连接外部系统来读取写入数据。 table source 访问存储在外部系统的数据（如数据库，KV数据库，消息队列，或文件系统）。 table sink 将数据输出到外部存储系统。 根据source 和 sink 的类型， 支持不同的格式如CSV, Parquet 或 ORC。


