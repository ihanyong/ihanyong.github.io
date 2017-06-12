### Cloud Native
Cloud Native is a style of applications development that encourages easy adoption of best practices in the areas of continuous delivery and value-driven development. A related discipline is that of building 12-ractor apps in which development practices are aligned with delivery and operations goals, for instance by useing declarative programming and management and monitoring. Spring Cloud facilitates these styles of development in a number of specific ways and the starting pont is a set of features that all components in a distributed system either need or need easy access to when requird.

Many of those features are covered by Spring Boot, which we build on in Spring Cloud. Some more are delivered by Spring
Cloud as tow libraries: Spring Cloud Context and Spring Cloud Commons. Spring Cloud Context provides utilities and speci
al services for the ApplicationContext of  Spring cloud application (bootstrap context, encryption, refresh scope and eveironment endpoints). Spring Cloud Commons is a set of abstractions and common classes used in different Spring Cloud im
plementations (eg.Spring Cloud Netflix vs. Spring Cloud Consul).

If you are getting an exception due to "Illegal key size" and you are useing Sun's JDK, you need to install the Java Cry
ptography Extension (JCE) Unilimited Strenght Jurisdiction Policy Files.

#### Spring Cloud Context: Application Context Services

Spring Boot has an opinionated view of how to build an application with Spring: for instance it has conventional locations for common configuration file, and endpoints for common management and monitoring tasks. Spring Cloud builds on top of that and adds a few features that porbably all components in a system would use or occasionally need.

##### The Bootstrap Application Context
A Spring Cloud application operates by creating a "bootstrap" context,  which is a parent context for the main application. Out of the box it is responsible for loading configuration properties from the external sources, and alse decryptiong properties in the lcoal external configuration files. The tow contexts share an Environment which is the source of external properties for any Spring application. Bootstrap properties are added whith high precedence, so they cannot be overridden by lcoal configuration, by default.

the boot strap context uses a different convention for locating external configuration than the main application context so instead of applicatin.yml (or .properties) you use bootstrap.yml, keeping the external configuration for bootstrap and main context nicely separate. Example:
bootstrap.yml
```
spring:
  application:
    name: foo
  cloud:
    config:
      uri:${SPRING_CONFIG_URI:httw://localhost:8888}
```

It is agood idea to set the spring.application.name if your application needs any application-specific configuration from the server.

You can disable the bootstrap process completely by setting spring.cloud .bootstrap.enabled=false )e.g. in System properties).

 
