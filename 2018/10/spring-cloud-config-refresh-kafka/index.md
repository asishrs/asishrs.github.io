# Spring Cloud Config Auto Refresh + Custom Kafka topic


<img class="cp t u fz ak" src="/images/posts/spring-kafka/spring-kafka.png" role="presentation"/>

[Spring Cloud Config](https://cloud.spring.io/spring-cloud-config/) is one of the best features that Spring provides as part of the framework. Spring Cloud Config allows your java application to follow Externalized configuration pattern which is must have if you are building microservices. Additionally, you can also enable the automatic config refresh in Spring Cloud Config so that all your components receive the latest configuration values when there is a change in the configuration. For automatic configuration delivery, Spring Cloud use Kafka or RabbitMQ messaging platforms. This article is going to explain how to define your own Kafka topics with Spring Cloud Config.

Using default configuration, Spring Cloud Config use predefined Kafka channel springCloudBusInput and springCloudBusOutput. Also, Kafka configuration expects you to provide the _zookeeper_ nodes using the option _spring.cloud.stream.kafka.binder.zkNodes_. This works well if you are using a Kafka broker that you manage or you have control. In that instance, your configuration looks like below.

```
spring:
 cloud:
  stream:
   kafka:
    binder:
     zkNodes: "<zookeeper-host>:<zookeeper-port>"
     brokers: "<broker-host>:<broker-port>"
```

What if you have to mention your own Kafka topics?
--------------------------------------------------

Above configuration won’t work if you are using a shared Kafka infrastructure which you cannot control, or you cannot create a topic dynamically on the Kafka. Luckily spring provides configuration options for you to pass custom Kafka topics and disable the auto topic creation.

Below are the options you can use for supply your topic names.

```
spring.cloud.stream.bindings.springCloudBusInput=<your\_custom\_input\_topic\_name>
spring.cloud.stream.bindings.springCloudBusOutput=<your\_custom\_output\_topic\_name>
```

You can disable the automatic topic creation using below option.

```
spring.cloud.stream.kafka.binder.autoCreateTopics=false
```

Complete configuration looks like below

```
spring.cloud.stream:
 bindings:
  springCloudBusInput:
   destination: <your\_custom\_kafka\_input\_topic>
   group: <optional group id>springCloudBusOutput:
   destination: <your\_custom\_kafka\_output\_topic>
 kafka.binder:
   autoCreateTopics: false
   brokers: <your\_kafka\_broker\_host>:<your\_kafka\_broker\_port>
```

Bonus — How can you use this with mutual authentication?
--------------------------------------------------------

Spring Cloud Stream supports configuration options for mutual authentication via SSL. Here is a sample configuration.

```
spring.cloud.stream:
 kafka.binder:
   autoCreateTopics: false
   brokers: <your\_kafka\_broker\_host>:<your\_kafka\_broker\_port>
   configuration:
     “\[security.protocol\]“: SSL
     “\[ssl.enabled\]“: true
     “\[ssl.truststore.location\]“: <path/truststore\_file>
     “\[ssl.truststore.password\]“: <your\_truststore\_password>
     “\[ssl.keystore.location\]“: <path/keystore\_file>
     “\[ssl.keystore.password\]“: <your\_keystore\_password>
     “\[ssl.key.password\]“: <your\_key\_password>
     “\[sasl.mechanism\]“: PLAIN
     “\[ssl.protocol\]“: TLSv1.2 // Check this with the Cert
     “\[ssl.enabled.protocols\]“: TLSv1.2 // Check this with the Cert
     “\[ssl.endpoint.identification.algorithm\]“: HTTPS
```

* * *

Reference
---------

* [Spring Cloud Config](https://cloud.spring.io/spring-cloud-config/)
* [Spring Cloud](https://spring.io/projects/spring-cloud)

