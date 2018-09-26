distributed streaming platform.


streaming platform has three key capabilities:
- publish and subscribe to streams of records, similar to a message queue or enterprise messaging system.
- Store streams of records in a fault-tolerant durable way.
- process streams of records as they occur.

generally used for tow broad classes of applications:
- building real-time streaming data pipelines that reliably get data between systems or applications.
- building real-time streaming applicatins that transform or react to the streams of data.

first a few concepts:
- Kafka is run as a cluster on one or more servers that can span multiple datacenters.
- The kafka cluster stores streams of records in categories called topics.
- Each record consists of a key, a value, and a timestamp.

Kafka has four core APIs:
- the producer api allows an application to publish a stream of records to one or more kafka topics.
- the consumer api allows an application to subscribe to one or more topics and process the stream of records produced to them.
- the streams api allows an application to act as a stream processor, consuming an input stream from one or more topics and producing and output stream to one or more output topics, effectively transforming the input streams to putput streams.
- the connector api allows buildig and running reusable producers or consumers that connect kafka topcis to existing applications  or data sysytems. for example, a connector to a relational databhase might capture every change to a table.


# Topics and Logs
A topic is a category or feed name to which records are published. Topics in kafka sre always multi-subscreiber; that is , a topic can have xero, one or many ocnsumers that subscribe to the data written to it.
...


# Distribution
# Geo-Replication
# Porducers
# Consumers
# Multi-tenancy
# Guarantees
# Kafka as a Messaging System
# Kafka as a Storage System
# Kafka for Stream Porcessing 




