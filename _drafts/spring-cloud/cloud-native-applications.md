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

 
#### Appliccation Context hierarchies

If you build an application context from SpringApplication or SpringApplicationBuilder, then the Bootstrap context is added as a parent to thant ocntext. It is a reature of Spring thant child contexts inherit property sources and profiles from th=eir parent, so the "main" application context will cotain additional property sourcfes, compared to building the same context without Spring Coud Config. The additionanl property sources are:

- "bootstrap": an optional CompositePropertySource appears with high priority if any PropertySourceLocatores are found in the Bootstrap context, and they have non-empty properties. An example would be properties from the the Spring Cloud Config Server. See below for instructions on how to contomize the contents of this property source.
- "applicationConfig": [classpath:bootstrap.yml]"(and frends if Spring profiles are active). If you have a bootstrap.yml (or properties) then those properties are used to configure the Bootstrap context, and then they get added to the child context when its parent is set. They have lower precedence than the application.yml (or properties) and any other property sources thant are added to the child as a normal part of the process of creating a Spring Boot application. See below for instructins on how to customize the contents of these property sources.

// TODO 




### Spring Cloud Commons: Common Abstractions

Patterns such as service discovery, load balancing and circuit breakers lead themselves to a common abstractions layer that can be ocnsumed by all Spring Cloud clients, independent of the implementation(e.g. discovery via Eureka or Consul).

#### @EnaleDiscouveryClient

Commons provides the @enaleDiscoveryClient annotation. This looks for implementations of the DoiscoveryClient interface via META-INF/spring.factories. Implementations of Discovery Client will add a configuration class to spring.factories under the org.springframework.cloud.client.discovery.EnableDiscoveryClient key. Examples of DiscoveryClient implementations: are Spring Cloud Netflix Eureka, Spring cloud Consul Discovery and Spring Cloud Zookeeper Discovery.

By default, implementations of DiscoveryClient will auto-register the local Spring Boot server with the remote discovery server. This can be disabled by setting autoRegister=false in @EnableDiscoveryClient.

#### ServiceRegistry 
Commons now provides a ServiceRegistry interface which provides methods like register(registration) and deregister(Registration) which allow you to provide custom registered services. Registration is a marker interface.

```
@Configuration
@EnableDiscoveryClient(autoRegister=false)
public class MyConfiguratin {
    private ServiceRegistry registry;

    public MyConfiguration(ServiceRegistry registry) {
        this.registry = registry;
    }
    
    // called via some external process, such as an event or a custom actuator endpoint
    public void register() {
        Registration registration = constructRegistration();
        this.registry.register(registratin);
    }
}

````

Each ServiceRegistry implementation has its own Registry implementation.

##### ServiceRegistry Auto-Registration
By default, the ServiceRegistry implementation will auto-register the running service. To disable that behavior, there are tow methods. You can set @EnableDiscoveryClient(autoRegister=false) to permanently disable auto-registration. You can also set spring.cloud.service-registry.auto-registration.enabled=false to disable the behavior via configuration.

##### Service Registry Actuator Endpoint
A /service-registry actuator endpoint is provided by Commons. This endpoint relys on a Registration bean in the Spring applicatoin Context. Calling /service-registry/instance-status via a GET will return the status of the Registration. A POST to the same endpoint with a Stirng body will change the status of the current Registration to the new value. Please see the documentation of the ServiceRegistry implementation you are useing for the allowed values for updating the staus and the values retured for the status.

##### Spring RestTemplate as a Load Balancer Client
RestTemplate can be automatically configured to use ribbon. To create a load balanced RestTemplate create a RestTemplate @Bean and use the @LoadBalanced qualifier.

warning
> A RestTemplate ean is no longer create via auto configuration. It must be created by individual applications.

```
@Configuration
public class MyConfiguration {
    @LoadBalanced
    @Bean
    RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

public class MyClass {
    @Autowired
    private RestTemplate restTemplate;
    
    public String doOtherStuff() {
       String results = restTemplate.getFroObject("http://****");
       return results;
    }
}
```
The URI needs to use a virtual host name (ie. service name, not a host name). The Ribbon client is used to create a full physical address. See RibbonAutoConfiguration for details of how the RestTemplate is set up.

##### Retrying Failed Requests
// TODO  
