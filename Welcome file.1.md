# Java Spring Boot - REST API and Kafka Message Publishing

WP Production: [https://www.codenow.com/docs/complex-examples/java-spring-boot/java-spring-boot-rest-api-and-kafka-message-publishing/](https://www.codenow.com/docs/complex-examples/java-spring-boot/java-spring-boot-rest-api-and-kafka-message-publishing/)

WP Staging: [https://m0e.6ae.myftpupload.com/wp-admin/post.php?action=edit&post=7070](https://m0e.6ae.myftpupload.com/wp-admin/post.php?action=edit&post=7070)

  

🕓 40 minutes

  

## What you’ll learn

How to setup your application for :

-   connecting to Kafka and publishing messages to its’ topic,
    
-   getting data from REST API,
    
-   providing data to REST API.
    

  

In this tutorial, we will create a simple java component with the Java Spring Boot scaffolder. We want to expose a single REST endpoint for getting client data. Client data is provided by another REST component client-data-db, so we need to configure a spring rest call for it. Any access to client data should be logged in the Kafka topic, so we need a Kafka client configuration as well.

  

![](https://lh3.googleusercontent.com/1NNSbVQFu35w3ZHEKjR30Adb7oiTosARoKlWi3a60ekgFqWavPEX7i2MmSUGoN1JnpG5YJT6F-ehvu-NWyVNAyaPA2mruBNYiAz9EyYYhqNWDqp9LzgLNc_XvmVw1jEengDUwwxC)

  

## Project source

This example project can be cloned from: [git@gitlab.factory.innobank.codenow.com](mailto:git@gitlab.factory.innobank.codenow.com):innobank/client-data-service.git

## Prerequisites

-   Prepare your local development environment for CodeNOW with Java Spring Boot.
    

-   Follow the tutorial instructions in [Java Spring Boot Local Development](https://www.codenow.com/docs/local-development-for-codenow/java-spring-boot-local-development/).
    

-   Run Apache Kafka locally.
    

-   Use docker compose as described in the section Docker compose and third-party tools of the [Java Spring Boot Local Development](https://www.codenow.com/docs/local-development-for-codenow/java-spring-boot-local-development/) tutorial.
    

-   Create a new component
    

-   For details see the section Prerequisites of the  [Java Spring Boot Local Development](https://www.codenow.com/docs/local-development-for-codenow/java-spring-boot-local-development/) tutorial.
    

## Steps

Open your IDE, import the created component and start coding:

  

-   Define the message payload. Here is an example of the Client, which is a simple POJO with basic client data:
    

-   generate getters and setters with your IDE
    

[java]

package io.codenow.client.data.service.model;

  

import java.time.LocalDate;

  

public class Client {

private String username;

private String firstname;

private String surname;

private LocalDate birthdate;

  

}

[/java]

  

-   Next prepare the configuration for the Kafka logging client:
    

-   Go to the Kafka administration console ([http://localhost:9000](http://localhost:9000) if using Kafdrop from our [Java Spring Local Development](https://www.codenow.com/docs/local-development-for-codenow/java-spring-boot-local-development/) manual.) and create a new topic client-logging
    
-   Add maven dependency to your pom.xml
    

[code]

<dependency>

<groupId>org.springframework.kafka</groupId>

<artifactId>spring-kafka</artifactId>

</dependency>

[/code]

  

-   For more details about spring-kafka, see: [https://spring.io/projects/spring-kafka](https://spring.io/projects/spring-kafka)
    
-   Now add the configuration for the Kafka template to your Application.java (package io.codenow.client.data.service):
    

[java]

### @Value("${kafka.broker.url}") private String kafkaBrokerUrl;

  

@Bean

public ProducerFactory<String, String> producerFactory() {

return new DefaultKafkaProducerFactory<>(producerConfigs());

}

  

@Bean

public Map<String, Object> producerConfigs() {

Map<String, Object> props = new HashMap<>();

props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, kafkaBrokerUrl);

props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);

props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);

// See https://kafka.apache.org/documentation/#producerconfigs for more properties

return props;

}

  

@Bean

public KafkaTemplate<String, String> kafkaTemplate() {

return new KafkaTemplate<String, String>(producerFactory());

}

[/java]

  
  

-   Next, create a new controller and put all the parts together
    

-   For more details about the spring REST controller, see: [https://spring.io/guides/gs/rest-service/](https://spring.io/guides/gs/rest-service/)
    

  

[java]

package io.codenow.client.data.service.controller;

  

import org.slf4j.Logger;

import org.slf4j.LoggerFactory;

import org.springframework.beans.factory.annotation.Value;

import org.springframework.kafka.core.KafkaTemplate;

import org.springframework.web.bind.annotation.GetMapping;

import org.springframework.web.bind.annotation.PathVariable;

import org.springframework.web.bind.annotation.RequestMapping;

import org.springframework.web.bind.annotation.RestController;

import org.springframework.web.reactive.function.client.WebClient;

  

import io.codenow.client.data.service.model.Client;

import reactor.core.publisher.Flux;

  

@RestController

@RequestMapping("/data")

public class ClientDataController {

private static final Logger LOG = LoggerFactory.getLogger(ClientDataController.class);

  

private String clientDataDBURL;

private String kafkaTopicName;

private String kafkaTopicKey;

private KafkaTemplate<String, String> kafkaTemplate;

  
  

public ClientDataController(@Value("${endpoint.client.data.db}") String clientDataDBURL, @Value("${kafka.topic.name}") String kafkaTopicName, KafkaTemplate<String, String> kafkaTemplate, @Value("${kafka.topic.key}") String kafkaTopicKey) {

super();

this.clientDataDBURL = clientDataDBURL;

this.kafkaTopicName = kafkaTopicName;

this.kafkaTemplate = kafkaTemplate;

this.kafkaTopicKey = kafkaTopicKey;

}

  

@GetMapping("/{username}")

private Flux<Client> getClientData(@PathVariable String username) {

LOG.info("Get data for username: {}", username);

kafkaTemplate.send(kafkaTopicName, kafkaTopicKey, username);

  

Flux<Client> clientFlux = WebClient.create().get().uri(clientDataDBURL + "/db/clients/" + username).retrieve()

.bodyToFlux(Client.class);

  

clientFlux.subscribe(client -> LOG.info(client.toString()));

return clientFlux;

  

}

}

[/java]

  
  

-   Last but not least, append the configuration for Kafka to config/application.yaml
    

-   Note that this configuration depends on your local development setup for Kafka and can differ case-by-case.
    
-   Make sure you follow yaml syntax (especially whitespaces)
    

[code]

endpoint:

client:

data:

db: http://client-data-db

kafka:

broker:

url: client-logging-kafka-kafka-brokers.managed-components:9092

topic:

name: client-logging

key: client-data-service

[/code]

  

-   Try to build and run the application in your IDE. After startup, you should be able to access your new controller’s swagger: http://localhost:8080/swagger/index.html
    

![](https://lh5.googleusercontent.com/atmNpc4-O7_Nb7utU8LzlBux01VBpBLb5ODsyWu6pSjgD_fFne7rZtFGVAfwrEiTkasCW2Slu0a1rFuzOg560yaL_1lf06REjKpLVFPYinxgQ2f1ujs5SPbXNoq5Ee3Jda3BPymb)

  
  

## What’s next?

If your code works in the local development, you are ready to push your changes to GIT and try to build and deploy your new component version to the CodeNOW environment. For more information see [Application Deployment](https://www.codenow.com/docs/administration-manuals/deploy-application/) and [Monitoring](https://www.codenow.com/docs/administration-manuals/deployment-monitoring/), just make sure to change the application.yaml properties from the local to the production setup.

-   Check [Get New Apache Kafka](https://www.codenow.com/docs/managed-components-administration/get-new-apache-kafka/) for setup in the CodeNOW environment.
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTkzNTI2ODk0OF19
-->