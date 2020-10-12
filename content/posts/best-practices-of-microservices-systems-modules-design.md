---
title: "Best practices of microservices systems modules design"
date: 2020-10-11
draft: false
---

## Approaches for working with microservices

So, microservices realize important modularity principles, leading to tangible benefits:
- Teams can work and scale independently.
- Microservices are small and focused, reducing complexity.
- Services can be internally changed or replaced without global impact.

Yet, it comes with its own set of drawbacks. It's not very clear how to design modules in microservices architecture in such way that can enforce this benefits.

Modularity in software development can be boiled down into three guiding principles:
- Strong encapsulation: hide implementation details inside components, leading to low coupling between different parts. Teams can work in isolation on decoupled parts of the system.
- Well-defined interfaces: you can’t hide everything (or else your system won’t do anything meaningful), so well-defined and stable APIs between components are a must. A component can be replaced by any implementation that conforms to the interface specification.
- Explicit dependencies: having a modular system means distinct components must work together. You’d better have a good way of expressing (and verifying) their relationships.

### Design modules

Creating good modules requires the same design rigor as creating good microservices. A module should model (part of) a single bounded context of the domain. Choosing microservice boundaries is an architecturally significant decision with costly ramifications when done wrong. Module boundaries in a modular application are easier to change. 

In many ways, modules in statically typed languages offer better constructs for well-defined interfaces. Calling a method through a typed interface exposed by another module is much more robust against changes than calling a REST endpoint on another microservice. REST+JSON is ubiquitous, but it is not the hallmark of well-typed interoperability in the absence of (compiler-checked) schemas. For example you can use Apache Avro to serialize/deserialize data and check data using the given schema from Kafka Schema Registry.

Many module systems allow you to express your dependencies on other modules. When these dependencies are violated, the module system will not allow it. Dependencies between microservices only materialize at run-time, leading to hard to debug systems.

Modules are natural units for code-ownership as well. Teams can be responsible for one or more modules in the system. The only thing shared with other teams is the public API of their modules.

Sharing within the modular application then happens through well-defined interfaces or messages between modules, not through a shared datastore. The big difference with microservices is that everything happens in-process. For modules you can choose eventual consistency or transaction use. For microservices, there is no choice: eventual consistency is a given and you need to adapt.

### Microservice Chassis pattern

When you start the development of an application you often spend a significant amount of time putting in place the mechanisms to handle cross-cutting concerns. Examples of cross-cutting concern include:

- Externalized configuration - includes credentials, and network locations of external services such as databases and message brokers
- Logging - configuring of a logging framework such as log4j or logback
- Health checks - a url that a monitoring service can “ping” to determine the health of the application
- Metrics - measurements that provide insight into what the application is doing and how it is performing
- Distributed tracing - instrument services with code that assigns each external request an unique identifier that is passed between services.

You need to have such kind of features for mitigating this problems:

- Creating a new microservice should be fast and easy
- When creating a microservice you must handle cross-cutting concerns such as externalized configuration, logging, health checks, metrics, service registration and discovery, circuit breakers. There are also cross-cutting concerns that are specific to the technologies that the microservices uses.

The major benefit of a microservice chassis is that you can quickly and easy get started with developing a microservice.

You need a microservice chassis for each programming language/framework that you want to use. This can be an obstacle to adopting a new programming language or framework.

### Externalized configuration pattern

An application typically uses one or more infrastructure and 3rd party services. Examples of infrastructure services include: a Service registry, a message broker and a database server. Examples of 3rd party services include: payment processing, email and messaging, etc.

You need to have such kind of features for mitigating this problems:

- A service must be provided with configuration data that tells it how to connect to the external/3rd party services. For example, the database network location and credentials
- A service must run in multiple environments - dev, test, qa, staging, production - without modification and/or recompilation
- Different environments have different instances of the external/3rd party services, e.g. QA database vs. production database, test credit card processing account vs. production credit card processing account

Solution is pretty straightforward. Externalize all application configuration including the database credentials and network location. On startup, a service reads the configuration from an external source, e.g. OS environment variables, etc.

## Using Gradle for creating modules

Gradle in few words:

### Gradle  multi-module project

Multi-project builds helps with modularization. It allows a person to concentrate on one area of work in a larger project, while Gradle takes care of dependencies from other parts of the project.

#### Create a root project
The first step is to create a folder for the new project and add a Gradle Wrapper to the project. If you use the Build Init plugin then the necessary settings and build scripts will also be added.

```bash
$ mkdir creating-multi-project-builds
$ cd creating-multi-project-builds
$ gradle init
```

Open the settings script. There will be a number of auto-generated comments which you can remove, leaving only:

`settings.gradle`
```groovy
rootProject.name = 'creating-multi-project-builds'
```

In a multi-project you can use the top-level build script (also known as the root project) to configure as much commonality as possible, leaving sub-projects to customize only what is necessary for that subproject.

`build.gradle`
```groovy
allprojects {
    repositories {
        jcenter() 1️⃣
    }
}
```

1️⃣ Add a Maven Repo for JCenter

## Add a library sub-project

```bash
$ mkdir greeting-library
```

`greeting-library/build.gradle`
```
plugins {
    id 'java'
}

dependencies {
    compile 'dependency:dependency:1.0'
}
```

Add liblary to the root project:

`settings.gradle`
```groovy
include 'greeting-library'
```

## Add a Java application sub-project

```bash
$ mkdir greeting-library
```

`greeting-library/build.gradle`
```
plugins {
    id 'java'        
    id 'application'
}

dependencies {
    compile project(':greeting-library') 1️⃣ 
}
```

1️⃣ Add `greeting-library` as a dependency for `greeter`

Add liblary to the root project:

`settings.gradle`
```groovy
include 'greeter'
```

See my Code example in (GitHub)[https://github.com/srcmaxim/microservices-modular-design].

Also checkout useful links for working with Gradle and modules:
- [Creating Multi-project Builds](https://guides.gradle.org/creating-multi-project-builds/)
- [Executing Multi-Project Builds](https://docs.gradle.org/current/userguide/intro_multi_project_builds.html)
- [Gradle liblary plugin](https://docs.gradle.org/current/userguide/java_library_plugin.html)
- [Gradle application plugin](https://docs.gradle.org/current/userguide/application_plugin.html)

### How to use Gradle + BOM

Some projects require that all dependencies 

`pom.xml`
```xml
<project>
  <modelVersion>4.0.0</modelVersion>
 
  <groupId>io.srcmaxim</groupId>
  <artifactId>dependencies</artifactId>
  <version>1</version>
  
  <dependencies>
    <dependency>
      <groupId>org.codehaus.groovy</groupId>
      <artifactId>groovy</artifactId>
      <version>3.0.6</version>
    </dependency>
  </dependencies>
</project>
```

`gradle.build`
```groovy
repositories {
    maven { url 'https://maven.pkg.github.com/<repo>/<org>' } 1️⃣
}

dependencies {
    compile platform('io.srcmaxim:dependencies:1') 2️⃣
    compile 'org.codehaus.groovy:groovy' 3️⃣ 
}
```

1️⃣ Add a Maven repository
2️⃣ Add .bom as a platform dependency
3️⃣ Use dependency without version. It will provided from .bom

Alternatively you can use [Spring Dependency Management Plugin](https://docs.spring.io/dependency-management-plugin/docs/current/reference/html/) for Gradle dependency management.

## Credits 
1 [Oreilly. Modules and microservices](https://www.oreilly.com/radar/modules-vs-microservices/)  
2 [Pattern: Microservice chassis](https://microservices.io/patterns/microservice-chassis.html)  
3 [Pattern: Externalized configuration](https://microservices.io/patterns/externalized-configuration.html)  
4 [Gradle. Creating Multi-project Builds](https://guides.gradle.org/creating-multi-project-builds/)  
5 [Code examples](https://github.com/srcmaxim/microservices-modular-design) 
