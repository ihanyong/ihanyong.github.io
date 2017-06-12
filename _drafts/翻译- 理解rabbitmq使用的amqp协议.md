---
layout: post
date:   2017-05-11 17:30:00 +0800
title: "Understanding AMQP, the protocol used by RabbitMQ"
categories: rabbitmq amqp spring
---
[origin url](https://spring.io/blog/2010/06/14/understanding-amqp-the-protocol-used-by-rabbitmq/)


RabbitMQ是一个轻量级, 可靠, 可扩展, 可移植的消息代理. 但他和Java开发者熟悉的其它消息代理不同,  它不是基于JMS, 而是基于一个平台无关的线路层协议:the Advanced Message Queuing Protocol (AMQP). 幸运的是已经有了一个Java客户端库, and SpringSource is working on first class Spring and Grails integration. 所以使用RabbitMQ时不用担心要处理底层的东西. 你甚至可以找到兼容JMS接口的AMQP客户端库. 会让使用JMS模式的Java开发者头疼的是AMQP在操作上与JMS还是有很大的不同.



### Exchange, queues, and bindings
和别的消息系统一样, AMQP是一个处理生产者和消费者的消息协议. 生产者生产消息, 消费者获取并处理消息. 消息代理(RabbitMQ)的工作是保证消息从生产者到达正确的消费者. 代理通过两个主要的组件来完成这件事. 如下图所示,代理如何连接生产者和消费者:
![]({{site.url/images/}})


Like any messaging system, AMQP is a message protocol that deals with publishers and consumers. The publishers produce the messages, the consumers pick them up and process them. It's the job of the message broker (such as RabbitMQ) to ensure that the messages from a publisher go to the right consumers. In order to do that, the broker uses two key components: exchanges and queues. The following diagram shows how they connect a publisher to a consumer:


### RPC

### Pub(lish)/Sub(scribe)

### Work distribution


