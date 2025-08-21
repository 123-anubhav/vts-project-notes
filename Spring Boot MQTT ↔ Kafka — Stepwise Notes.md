# Spring Boot MQTT ↔ Kafka — Stepwise Notes

**Goal:** show how a Spring Boot app can publish/subscribe to MQTT, and how MQTT messages can be forwarded into Kafka (two patterns: application-bridge and broker-bridge). Includes code, configuration, Docker-compose for local testing, and troubleshooting tips.

---

## Table of contents

1. Overview & architecture
2. Prerequisites
3. Pattern A — Spring Boot *acts as* MQTT publisher/subscriber and forwards to Kafka (recommended for app-level control)

   * Maven dependencies
   * `application.yml`
   * `MqttConfig` (inbound + outbound)
   * `KafkaProducerConfig`
   * Controller to publish to MQTT
   * Service that receives MQTT messages and writes to Kafka
4. Pattern B — Broker-level bridge (e.g., Kafka Connect / broker plugin)

   * When to use
   * Example connector snippet (Kafka Connect MQTT Source)
5. Docker Compose (Mosquitto + Zookeeper + Kafka) for local testing
6. Test steps & quick examples (mosquitto\_pub/sub, curl)
7. QoS, serialization, and security notes
8. Troubleshooting & tips

---

# 1. Overview & architecture

Two common flows you'll implement:

* **App-bridge (preferred for app logic):** Spring Boot subscribes to MQTT topics. When a message arrives, the same Spring Boot app sends that payload into Kafka (using `KafkaTemplate`). This gives you full control over transformations, filtering, headers, retries, schema conversion, logging, and async batching.

* **Broker-bridge (connector-level):** Use a connector/bridge that runs in the broker ecosystem (e.g., Kafka Connect MQTT Source, or a Mosquitto plugin) that reads MQTT topics and writes directly into Kafka topics. This is useful for lightweight forwarding without application code.

This document focuses on **Pattern A** with runnable code, and then briefly covers Pattern B.

---

# 2. Prerequisites

* Java 17+ (or Java 11 if your Spring Boot version requires it)
* Maven (or Gradle)
* Local test brokers: Mosquitto (MQTT) and Kafka + Zookeeper (we provide a docker-compose snippet)
* Spring Boot familiarity (controllers, `@Configuration`, and DI)

---

# 3. Pattern A — Spring Boot subscribes to MQTT and forwards to Kafka

## 3.1 Maven dependencies (pom.xml minimal snippet)

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>

  <!-- Spring Integration MQTT (uses Eclipse Paho internally) -->
  <dependency>
    <groupId>org.springframework.integration</groupId>
    <artifactId>spring-integration-mqtt</artifactId>
  </dependency>

  <!-- Spring for Apache Kafka -->
  <dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
  </dependency>

  <!-- Optional: Jackson for JSON -> String conversions etc. -->
  <dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
  </dependency>
</dependencies>
```

> Tip: If you prefer the raw Eclipse Paho client, you can use `org.eclipse.paho:org.eclipse.paho.client.mqttv3` and implement callbacks yourself — `spring-integration-mqtt` is higher level and recommended.

## 3.2 `application.yml` (example)

```yaml
mqtt:
  url: tcp://localhost:1883
  clientId: spring-boot-mqtt
  subscribeTopic: sensors/+/data
  defaultPublishTopic: sensors/commands

spring:
  kafka:
    bootstrap-servers: localhost:9092
```

## 3.3 MQTT + Spring Integration config (`MqttConfig.java`)

```java
package com.example.mqttkafka.config;

import org.eclipse.paho.client.mqttv3.MqttConnectOptions;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.annotation.ServiceActivator;
import org.springframework.integration.channel.DirectChannel;
import org.springframework.integration.core.MessageProducer;
import org.springframework.integration.mqtt.core.DefaultMqttPahoClientFactory;
import org.springframework.integration.mqtt.core.MqttPahoClientFactory;
import org.springframework.integration.mqtt.inbound.MqttPahoMessageDrivenChannelAdapter;
import org.springframework.integration.mqtt.outbound.MqttPahoMessageHandler;
import org.springframework.integration.mqtt.support.DefaultPahoMessageConverter;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.MessageHandler;

@Configuration
public class MqttConfig {

    @Value("${mqtt.url}")
    private String mqttUrl;

    @Value("${mqtt.clientId}")
    private String clientId;

    @Value("${mqtt.subscribeTopic}")
    private String subscribeTopic;

    @Bean
    public MqttPahoClientFactory mqttClientFactory() {
        DefaultMqttPahoClientFactory factory = new DefaultMqttPahoClientFactory();
        MqttConnectOptions options = new MqttConnectOptions();
        options.setServerURIs(new String[]{mqttUrl});
        // options.setUserName("user");
        // options.setPassword("pass".toCharArray());
        factory.setConnectionOptions(options);
        return factory;
    }

    // Inbound channel — messages from MQTT will come here
    @Bean
    public MessageChannel mqttInputChannel() {
        return new DirectChannel();
    }

    @Bean
    public MessageProducer inbound() {
        // clientId + "_in" ensures unique client id
        MqttPahoMessageDrivenChannelAdapter adapter =
                new MqttPahoMessageDrivenChannelAdapter(clientId + "_in", mqttClientFactory(), subscribeTopic);
        adapter.setCompletionTimeout(5000);
        adapter.setConverter(new DefaultPahoMessageConverter());
        adapter.setQos(1);
        adapter.setOutputChannel(mqttInputChannel());
        return adapter;
    }

    // Outbound channel — used by our Controller/service to publish to MQTT
    @Bean
    public MessageChannel mqttOutboundChannel() {
        return new DirectChannel();
    }

    @Bean
    @ServiceActivator(inputChannel = "mqttOutboundChannel")
    public MessageHandler mqttOutbound() {
        MqttPahoMessageHandler messageHandler = new MqttPahoMessageHandler(clientId + "_out", mqttClientFactory());
        messageHandler.setAsync(true);
        // Default topic (can be overridden by header when sending)
        messageHandler.setDefaultTopic("sensors/commands");
        return messageHandler;
    }
}
```

## 3.4 Kafka producer config (`KafkaProducerConfig.java`)

```java
package com.example.mqttkafka.config;

import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.common.serialization.StringSerializer;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.core.DefaultKafkaProducerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.core.ProducerFactory;

import java.util.HashMap;
import java.util.Map;

@Configuration
public class KafkaProducerConfig {

    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;

    @Bean
    public Map<String, Object> producerConfigs() {
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        return props;
    }

    @Bean
    public ProducerFactory<String, String> producerFactory() {
        return new DefaultKafkaProducerFactory<>(producerConfigs());
    }

    @Bean
    public KafkaTemplate<String, String> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
}
```

## 3.5 Controller to publish messages to MQTT (`MqttController.java`)

```java
package com.example.mqttkafka.web;

import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.integration.support.MessagingTemplate;
import org.springframework.messaging.Message;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class MqttController {

    private final MessagingTemplate messagingTemplate;

    public MqttController(@Qualifier("mqttOutboundChannel") MessageChannel mqttOutboundChannel) {
        this.messagingTemplate = new MessagingTemplate(mqttOutboundChannel);
    }

    @PostMapping("/api/mqtt/publish")
    public String publish(@RequestParam String topic, @RequestBody String payload) {
        Message<String> message = MessageBuilder.withPayload(payload)
                .setHeader("mqtt_topic", topic) // optional custom header
                .setHeader("mqtt_receivedTopic", topic) // Paho uses keys like this sometimes; can set proper header via MqttHeaders.TOPIC
                .build();

        // send will go through mqttOutbound() ServiceActivator
        messagingTemplate.send(message);
        return "ok";
    }
}
```

> Note: You can also use `MqttHeaders.TOPIC` (from `org.springframework.integration.mqtt.support.MqttHeaders`) as a header key to override target topic when using `MqttPahoMessageHandler`.

## 3.6 Service: receive MQTT messages and push to Kafka (`MqttKafkaBridgeService.java`)

```java
package com.example.mqttkafka.service;

import org.springframework.integration.annotation.ServiceActivator;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.messaging.Message;
import org.springframework.stereotype.Service;

@Service
public class MqttKafkaBridgeService {

    private final KafkaTemplate<String, String> kafkaTemplate;

    public MqttKafkaBridgeService(KafkaTemplate<String, String> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    // inbound channel name must match mqttInputChannel bean
    @ServiceActivator(inputChannel = "mqttInputChannel")
    public void handleMqttMessage(Message<?> message) {
        // payload usually a byte[] or String depending on converter
        String payload = message.getPayload().toString();

        // mqtt topic is available in header "mqtt_receivedTopic"
        Object topic = message.getHeaders().get("mqtt_receivedTopic");
        // You can route to different kafka topics depending on mqtt topic
        kafkaTemplate.send("mqtt-to-kafka-topic", payload);
    }
}
```

**That's it:** with this setup, incoming MQTT messages on `sensors/+/data` are delivered to the `mqttInputChannel`, processed by `MqttKafkaBridgeService` and forwarded into Kafka.

---

# 4. Pattern B — Broker-level bridge (when to use and a short example)

Use a broker-level bridge when you want a thin forwarder without application logic. Two common options:

* **Kafka Connect MQTT Source connector** (runs in Kafka Connect, reads MQTT topics and writes Kafka records). Good for simple ingestion pipelines and you can configure topics, converters, and transforms.
* **Broker plugin** (e.g., Mosquitto plugin) that writes to Kafka.

### Example Kafka Connect MQTT Source (conceptual)

Connector config (JSON) — this runs in Kafka Connect and subscribes to MQTT topics and writes to Kafka topics:

```json
{
  "name": "mqtt-source",
  "config": {
    "connector.class": "io.confluent.connect.mqtt.MqttSourceConnector",
    "tasks.max": "1",
    "mqtt.server.uri": "tcp://mosquitto:1883",
    "mqtt.topics": "sensors/#",
    "kafka.topic": "mqtt-to-kafka-topic",
    "confluent.topic.bootstrap.servers": "kafka:9092"
  }
}
```

> Connector class names and configuration keys depend on the connector implementation (Confluent, Lenses, etc.). Use the connector docs for exact keys.

---

# 5. Docker Compose for local testing (Mosquitto + Zookeeper + Kafka)

```yaml
version: '3.8'
services:
  mosquitto:
    image: eclipse-mosquitto:2.0
    ports:
      - "1883:1883"
      - "9001:9001"

  zookeeper:
    image: confluentinc/cp-zookeeper:7.2.1
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka:
    image: confluentinc/cp-kafka:7.2.1
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    ports:
      - "9092:9092"
```

> Quick note: your Docker networking for Kafka may need `KAFKA_ADVERTISED_LISTENERS` tuned for your host. Use local host mapping during development.

---

# 6. Test steps & quick commands

1. Start Docker compose: `docker compose up -d`.
2. Start the Spring Boot app (it will connect to mqtt and kafka as configured).
3. To publish directly to MQTT (test producer):

```bash
mosquitto_pub -h localhost -t sensors/room1/data -m '{"temp":23}'
```

4. To view that message on the MQTT side (debug):

```bash
mosquitto_sub -h localhost -t 'sensors/#' -v
```

You should see the message appear in the MQTT subscriber terminal and — if your app is running — the message will be forwarded to Kafka. To verify Kafka ingestion, use a simple Kafka console consumer:

```bash
# Create topic if needed
kafka-topics --create --topic mqtt-to-kafka-topic --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1

# Consume
kafka-console-consumer --bootstrap-server localhost:9092 --topic mqtt-to-kafka-topic --from-beginning
```

5. To publish to MQTT via the Spring Boot REST API example:

```bash
curl -X POST 'http://localhost:8080/api/mqtt/publish?topic=sensors/room1/commands' -d 'turn_on'
```

---

# 7. QoS, serialization, and security notes

* **QoS:** MQTT supports 0,1,2 — choose based on reliability needs. Use QoS=1 or 2 for important messages.
* **Message format:** Convert MQTT payloads to a canonical format before writing to Kafka (JSON, Avro, Protobuf). If you need schema enforcement, use Avro + Schema Registry.
* **Idempotency / dedup:** If the same payload may be redelivered, ensure downstream consumers can handle duplicates or add dedup keys.
* **Security:** Use TLS for MQTT (`ssl://...`), enable MQTT authentication, and enable Kafka authentication (SASL/SSL) if moving to production.
* **Back-pressure:** When forwarding high throughput, buffer/async send to Kafka and handle failures/retries — `KafkaTemplate` supports async send and callbacks.

---

# 8. Troubleshooting & tips

* If Spring Boot cannot connect to MQTT: check the broker URL, clientId uniqueness, and any firewall rules. Use mosquitto\_sub to validate the broker.
* If Kafka send fails: check `bootstrap-servers` and container advertised listeners. Also inspect Kafka broker logs.
* For local Docker-based Kafka, advertised listeners often cause connectivity issues — set to `PLAINTEXT://localhost:9092` for local dev.
* Use structured logging and include mqtt topic headers when forwarding to Kafka for traceability.

---

# 9. Alternatives & scaling

* For high-throughput ingestion at scale, prefer a Kafka Connect MQTT Source so ingestion is decoupled from application instances.
* For filtering/transformation, keep business logic in the application-bridge approach and consider adding a lightweight stream processing (Kafka Streams or ksqlDB) downstream.

---

# 10. Next steps I can help with

* Provide a ready-to-run sample GitHub repo with full project files (pom, complete classes, Docker setup).
* Convert the message payload to Avro and show Schema Registry examples.
* Show how to implement retries and dead-lettering when Kafka is unavailable.


Done — I created detailed step-by-step notes (code, `application.yml`, Docker Compose, test commands, and troubleshooting) in a canvas titled **Spring Boot MQTT ↔ Kafka — Stepwise Notes**. Open the document (on the right) and tell me if you want:

* a runnable GitHub-ready project (I can generate full `pom.xml` and all class files),
* the same example but using Gradle, or
* a Pattern B connectors walkthrough (Kafka Connect config + how to run it).


*End of notes.*
