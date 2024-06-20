---
layout: post
title:  "Kafka Integration with SpringBoot"
date:   2024-06-10 21:03:36 +0530
categories: SpringBoot Java Kafka
---
Kafka Integration with SpringBoot

## Kafka docker-compose file

- kafka
- zookeeper
- kafka ui

```yml
version: '2'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - 52181:2181

  kafka:
    image: confluentinc/cp-kafka:7.0.1
    ports:
      - "9092:9092"
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_INTERNAL:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092,PLAINTEXT_INTERNAL://kafka:29092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1

  kafka-admin:
    image: provectuslabs/kafka-ui
    container_name: kafka-admin
    ports:
      - "9091:8080"
    restart: always
    depends_on:
      - kafka
      - zookeeper
    environment:
      - KAFKA_CLUSTERS_0_NAME=local
      - KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS=kafka:29092
      - KAFKA_CLUSTERS_0_ZOOKEEPER=zookeeper:2181
```


**pom.xml**<br>

dependencies

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka-test</artifactId>
    <scope>test</scope>
</dependency>
```

### Producer Application

**application.properties**

```properties
spring.kafka.topic.name=order_topics

spring.kafka.consumer.bootstrap-servers=localhost:9092
spring.kafka.consumer.group-id=stock
spring.kafka.consumer.auto-offset-reset=earliest
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.springframework.kafka.support.serializer.JsonDeserializer
spring.kafka.consumer.properties.spring.json.trusted.packages=*

spring.kafka.producer.bootstrap-servers=localhost:9092
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.value-serializer=org.springframework.kafka.support.serializer.JsonSerializer

```

**KafkaTopicConfig.class**

```java
@Configuration
public class KafkaTopicConfig {

    @Value("${spring.kafka.topic.name}")
    private String topicName;

    @Bean
    public NewTopic topic(){
        return TopicBuilder.name(topicName)
                .build();
    }

}
```

> There should be a OrderEvent class to send message as an object


```java
public class OrderEvent {
    private String message;
    private String status;
    private Order order;
    //constructor
    //setters and getters
}
```

**OrderProducer.class** <br>

We should pass OrderEvent object to sendMessage method, event is set as payload`.withPayload(event)`.<br>
topic name is set as header `.setHeader(KafkaHeaders.TOPIC, topic.name())` 

```java
@Service
@Slf4j
public class OrderProducer {

    private final NewTopic topic;
    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    private OrderProducer(NewTopic topic, KafkaTemplate<String, OrderEvent> kafkaTemplate){
        this.topic = topic;
        this.kafkaTemplate = kafkaTemplate;
    }

    public void sendMessage(OrderEvent event){
        // create message
        Message<OrderEvent> message = MessageBuilder
                .withPayload(event)
                .setHeader(KafkaHeaders.TOPIC, topic.name())
                .build();

        kafkaTemplate.send(message);
    }

}
```

**OrderService.class** <br>

OrderProducer is autowired to OrderService.class to send message


```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderProducer orderProducer;

    public String placeOrder(Order order){

        order.setOrderId(UUID.randomUUID().toString());

        OrderEvent orderEvent = new OrderEvent();
        orderEvent.setStatus("PENDING");
        orderEvent.setMessage("Order is pending state");
        orderEvent.setOrder(order);

        orderProducer.sendMessage(orderEvent);

        return "Order placed...";
    }

}
```

### Consumer Application

consumer properties can be just deserializer, but I want to give deserializer and serializers
 
**application.properties**

```properties
spring.kafka.topic.name=order_topics

spring.kafka.consumer.bootstrap-servers=localhost:9092
spring.kafka.consumer.group-id=stock
spring.kafka.consumer.auto-offset-reset=earliest
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.springframework.kafka.support.serializer.JsonDeserializer
spring.kafka.consumer.properties.spring.json.trusted.packages=*

spring.kafka.producer.bootstrap-servers=localhost:9092
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.value-serializer=org.springframework.kafka.support.serializer.JsonSerializer

```

**OrderConsumer.class**

OrderConsumer is a service, `@KafkaListener` annotation should be used for consume method. OrderEvent should be passed to consume method.

```java
@Service
public class OrderConsumer {


    @KafkaListener(
            topics = "${spring.kafka.topic.name}",
            groupId = "${spring.kafka.consumer.group-id}"
    )
    public void consume(OrderEvent orderEvent){
        System.out.println(orderEvent.getOrder().getOrderId());
        System.out.println(orderEvent.getOrder().getName());
        System.out.println(orderEvent.getOrder().getQty());
    }

}
```