kafka的设计.md
# 动机
处理大公司需要的实时数据流。 
一些用例
- 以高吞吐量支持大规模事件流（如实时日志聚合）
- 优雅地处理大规模数据的积压， 以支持从离线系统加载定期数据
- 低延迟，以处理传统的消息用例
- 分片、分布式、实时处理
- 容错性

# 持久化


curl -X POST 'http://127.0.0.1:8848/nacos/v1/ns/instance?serviceName=nacos.naming.serviceName&ip=20.18.7.10&port=8080'

curl -X POST 'http://127.0.0.1:8848/nacos/v1/ns/instance?serviceName=nacos.naming.serviceName&ip=20.18.7.10&port=8080'