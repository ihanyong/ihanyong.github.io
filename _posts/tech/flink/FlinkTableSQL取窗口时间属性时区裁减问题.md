FlinkTableSQL取窗口时间属性时区裁减问题.md

DataStreamTimeWindowPropertyCollector.collector()




TimeWindowPropertyCollector  line 58    SqlFunctions.internalToTimestamp(windowEnd))

- [FLINK-4691] [table] Rework window property extraction. Fabian Hueske 2016/10/26 7:08 添加了时区裁减
- [FLINK-6697] [table] Add support for group window ROWTIME to batch SQL & Table API. Fabian Hueske 2017/8/7 5:55  
- [FLINK-7337] [table] Refactor internal handling of time indicator attributes. Fabian Hueske* 2017/8/4 8:20 [FLINK-7337] [table] Efficient handling of rowtime timestamps twalthr 2017/8/12 19:51      添加并取消了TUMBLE_ROWTIME时区裁减



https://issues.apache.org/jira/browse/FLINK-4691
https://issues.apache.org/jira/browse/FLINK-6697
https://issues.apache.org/jira/browse/FLINK-7337
https://issues.apache.org/jira/browse/FLINK-11010



http://apache-flink-user-mailing-list-archive.2336050.n4.nabble.com/Timestamp-conversion-problem-in-Flink-Table-SQL-td25259.html#a25265


http://mail-archives.apache.org/mod_mbox/flink-user/201711.mbox/%3C351FD9AB-7A28-4CE0-BD9C-C2A15E5372D6@163.com%3E




时区的问题




