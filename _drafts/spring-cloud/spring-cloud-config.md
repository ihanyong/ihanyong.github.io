Spring Cloud Config provides server and client side support for externalized configuartion in a distributed system. With the Config Server you have a central place to manage external properties for applications across all environments. The concepts on both client and server map identically to the Spring Environment and PropertuSource abstractions, so they fit very well with Spring applications, but can be used with any application running in any language. As an application moves through the deployment pipeline from dev to test and into production you can manage the configuration between those environments and be certain that applicaions have everything they need to run when they migrate. The default implementation of the server storage backend uses git so it easily supports labelled versions of configuration environments, as well as being accessible to a wide range of tooling for managing the content. It is easy to add alternative implementations and plug them in with Spring Configuration.

#### Quick Start
Start the server:
```
$ cd spring-cloud-config-server
$ ../mvnw spring-boot:run
```
The server is a Spring Boot application so you can run it from IDE instead if you prefer (the main class is ConfigServerApplicaiton). Then try out a client:
```
$ curl localhost:8888/foo/development
{"name":"development", "label":"master", "propertySoucrces":[
  {"name":"https://github.com/scratches/config-repo/foo-development.properties","source":{"bar":"spam"}},
  {"name":"https://github.com/scratches/config-repo/foo.properties","source":{"foo":"bar"}}
 ]}
```
 
