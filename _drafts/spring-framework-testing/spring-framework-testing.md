spring-framework-testing.md

采用测试驱动开发是被很多团队推荐的。Spring 也是对单元测试、集成测试进行了支持。 Spring 的开发团队发现正确地使用IoC能让单元和集成测试变得更简单。

# Spring Testing 简介
测试是企业软件开发必不可少的部分。这里主要聚焦在IoC原则对单元测试的加益和Spring框架对集成测试的支持。

# 单元测试
相比于传统的JavaEE开发，依赖注入使得代码更少依赖容器。 POJOs让应用程序


# 这两天的调查学习
## Junit & TestNG
相比于Junit， TestNG支持的功能更多，理念也更新。 但是现在使用的 Spring-boot 版本内置的测试框架是Junit4， 所以单元测试工具依然使用Junit4就好，毕竟已经够用。
## Mockito
对于 Mock, 各有各自的说法了， 还有Jmock等其它可用的框架， 但基本是不采用 __stubing__的方式了。
 