**comprehensive end-to-end Kafka Streams notes** for a **Spring Boot project**, covering **concepts, setup, why/when to use, code examples, and best practices**.

---

# **Kafka Streams End-to-End Notes with Spring Boot**

---

## **1. Introduction: What is Kafka Streams?**

* **Kafka Streams** is a **client library** for building **real-time stream processing applications**.
* It allows you to process data stored in **Kafka topics** and output results to **Kafka topics** or other systems.
* **Part of Apache Kafka** ecosystem: lightweight, embeddable, fault-tolerant, scalable.

**Key concepts:**

1. **Streams**: Continuous flow of data from topics.
2. **Tables (KTable)**: Materialized view of a topic (latest state).
3. **Processor API / DSL**: Two ways to build stream processing:

   * **DSL**: High-level operations (map, filter, groupBy, aggregate).
   * **Processor API**: Low-level custom processing logic.
4. **Topology**: Graph of stream processors and data flow.

---

## **2. Why Kafka Streams?**

* Real-time processing of data.
* Fault-tolerant and scalable.
* Integrates easily with Kafka: no separate cluster needed.
* Useful for:

  * Data transformation
  * Aggregation / counting
  * Filtering and enrichment
  * Join streams and tables

---

## **3. When to Use Kafka Streams?**

| Use Case                   | Why Kafka Streams                  |
| -------------------------- | ---------------------------------- |
| Counting events per minute | Real-time aggregation              |
| Filtering user activity    | Real-time filtering                |
| Joining two streams        | Stream enrichment                  |
| Stateful processing        | Maintain counts, windows, sessions |
| Event-driven microservices | Lightweight, embedded in app       |

---

## **4. Kafka Streams Concepts**

### **4.1 Stream vs Table**

* **KStream**: Unbounded record stream (event-driven).
* **KTable**: Table view (latest state), supports updates and deletions.

### **4.2 Key Functions in DSL**

* `map()`, `filter()`, `flatMap()`: Transform stream.
* `groupByKey()`, `aggregate()`, `count()`, `reduce()`: Stateful operations.
* `join()`: Join streams or stream with table.
* `windowedBy()`: Time-based aggregation.

### **4.3 Topology**

* Represents processing graph.
* Built automatically in DSL, or manually in Processor API.

### **4.4 Serdes**

* Serializer/Deserializer for converting objects to bytes.
* Common: `StringSerde`, `LongSerde`, `JsonSerde`.

---

## **5. Spring Boot + Kafka Streams Setup**

### **5.1 Maven Dependencies**

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka</artifactId>
    </dependency>

    <dependency>
        <groupId>org.apache.kafka</groupId>
        <artifactId>kafka-streams</artifactId>
    </dependency>
</dependencies>
```

---

### **5.2 application.properties**

```properties
spring.kafka.bootstrap-servers=localhost:9092
spring.kafka.streams.application-id=kafka-streams-demo
spring.kafka.streams.default-key-serde=org.apache.kafka.common.serialization.Serdes$StringSerde
spring.kafka.streams.default-value-serde=org.apache.kafka.common.serialization.Serdes$StringSerde
spring.kafka.consumer.auto-offset-reset=earliest
```

---

## **6. Kafka Streams Configuration in Spring Boot**

```java
package com.example.demo.config;

import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.kstream.KStream;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class KafkaStreamsConfig {

    @Bean
    public KStream<String, String> kStream(StreamsBuilder builder) {
        KStream<String, String> stream = builder.stream("input-topic");
        stream.filter((key, value) -> value.contains("important"))
              .mapValues(value -> value.toUpperCase())
              .to("output-topic");
        return stream;
    }
}
```

**Explanation:**

* Reads from `input-topic`.
* Filters messages containing “important”.
* Converts value to uppercase.
* Writes to `output-topic`.

---

## **7. Kafka Streams Operations Examples**

### **7.1 Map & Filter**

```java
stream.mapValues(value -> value.toLowerCase())
      .filter((key, value) -> value.contains("spring"))
      .to("processed-topic");
```

### **7.2 Grouping & Counting**

```java
stream.groupByKey()
      .count()
      .toStream()
      .to("count-topic");
```

### **7.3 Join Streams**

```java
KStream<String, String> stream1 = builder.stream("topic1");
KStream<String, String> stream2 = builder.stream("topic2");

stream1.join(stream2,
    (v1, v2) -> v1 + "-" + v2
).to("joined-topic");
```

### **7.4 Windowed Aggregation**

```java
stream.groupByKey()
      .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(5)))
      .count()
      .toStream()
      .to("windowed-count-topic");
```

---

## **8. Serdes for Custom Objects**

```java
import org.apache.kafka.common.serialization.Serdes;
import org.springframework.kafka.support.serializer.JsonSerde;

@Bean
public KStream<String, MyObject> objectStream(StreamsBuilder builder) {
    KStream<String, MyObject> stream = builder.stream(
        "input-topic", Consumed.with(Serdes.String(), new JsonSerde<>(MyObject.class))
    );
    stream.to("output-topic", Produced.with(Serdes.String(), new JsonSerde<>(MyObject.class)));
    return stream;
}
```

---

## **9. Topology Example**

```java
@Bean
public KafkaStreams kafkaStreams(StreamsBuilder builder) {
    KStream<String, String> stream = builder.stream("input-topic");
    stream.filter((k, v) -> v.contains("hello")).to("output-topic");

    Topology topology = builder.build();
    KafkaStreams streams = new KafkaStreams(topology, streamsConfig());
    streams.start();
    return streams;
}
```

---

## **10. Processor API Example**

```java
builder.stream("input-topic")
       .process(() -> new AbstractProcessor<String, String>() {
           @Override
           public void process(String key, String value) {
               System.out.println("Key=" + key + ", Value=" + value);
               context().forward(key, value.toUpperCase());
           }
       })
       .to("output-topic");
```

---

## **11. Kafka Streams Best Practices**

1. **Always use proper Serdes** (String, Long, JSON).
2. **Use KTable for stateful operations** (aggregations, joins).
3. **Windowed aggregation** is key for time-based analytics.
4. **Handle exceptions** carefully; use `try-catch` or `deserialization.exception.handler`.
5. **Monitor application** with `KafkaStreams#metrics()`.
6. **Compact topics** for KTables to save space.
7. **Partitioning** matters: same key → same partition → consistent processing.

---

## **12. Practice Plan for Kafka Streams**

1. Setup Kafka locally (`zookeeper + kafka-server`).
2. Create `input-topic` and `output-topic`.
3. Build a Spring Boot stream to filter messages.
4. Implement `mapValues()` transformation.
5. Count messages per key (`groupByKey() + count()`).
6. Join two topics.
7. Implement time-windowed aggregation.
8. Use KTable for latest state.
9. Serialize custom object with `JsonSerde`.
10. Monitor stream application metrics.

---

✅ **Summary**

* Kafka Streams = real-time stream processing library.
* Can be **embedded in Spring Boot applications**.
* Supports **DSL** and **Processor API**.
* Provides **stateful processing, joins, windowed operations, and aggregation**.
* Use **KStream** for event streams, **KTable** for stateful table-like data.

---

**ready-to-run Spring Boot Kafka Streams project** with:

* Input/Output topics
* Example stream processing
* Aggregation, join, and windowed operations
* JSON Serde support

Perfect! Here’s a **ready-to-run Spring Boot Kafka Streams project structure** with **example stream processing, aggregation, joins, windowed operations, and JSON Serde support**.

---

# **Spring Boot Kafka Streams Project**

---

## **1. Project Structure**

```
kafka-streams-demo/
├── src/main/java/com/example/demo/
│   ├── DemoApplication.java
│   ├── config/
│   │   └── KafkaStreamsConfig.java
│   ├── model/
│   │   └── UserEvent.java
│   ├── service/
│   │   └── StreamProcessingService.java
├── src/main/resources/
│   └── application.properties
├── pom.xml
```

---

## **2. pom.xml**

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" ...>
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>kafka-streams-demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>Kafka Streams Demo</name>

    <dependencies>
        <!-- Spring Boot Starter -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <!-- Spring Kafka -->
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka</artifactId>
        </dependency>

        <!-- Kafka Streams -->
        <dependency>
            <groupId>org.apache.kafka</groupId>
            <artifactId>kafka-streams</artifactId>
        </dependency>

        <!-- JSON Serde -->
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka</artifactId>
        </dependency>
    </dependencies>
</project>
```

---

## **3. application.properties**

```properties
spring.kafka.bootstrap-servers=localhost:9092
spring.kafka.streams.application-id=kafka-streams-demo
spring.kafka.streams.default-key-serde=org.apache.kafka.common.serialization.Serdes$StringSerde
spring.kafka.streams.default-value-serde=org.apache.kafka.common.serialization.Serdes$StringSerde
spring.kafka.consumer.auto-offset-reset=earliest
```

---

## **4. DemoApplication.java**

```java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

---

## **5. Model: UserEvent.java**

```java
package com.example.demo.model;

public class UserEvent {
    private String userId;
    private String eventType;
    private long timestamp;

    public UserEvent() {}

    public UserEvent(String userId, String eventType, long timestamp) {
        this.userId = userId;
        this.eventType = eventType;
        this.timestamp = timestamp;
    }

    // Getters and setters
}
```

---

## **6. KafkaStreamsConfig.java**

```java
package com.example.demo.config;

import com.example.demo.model.UserEvent;
import org.apache.kafka.common.serialization.Serdes;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.support.serializer.JsonSerde;
import org.apache.kafka.streams.kstream.*;
import org.apache.kafka.streams.StreamsBuilder;

@Configuration
public class KafkaStreamsConfig {

    @Bean
    public KStream<String, UserEvent> kStream(StreamsBuilder builder) {
        JsonSerde<UserEvent> userEventSerde = new JsonSerde<>(UserEvent.class);

        KStream<String, UserEvent> stream = builder.stream("input-topic",
                Consumed.with(Serdes.String(), userEventSerde));

        // Filter events
        stream.filter((key, value) -> "LOGIN".equals(value.getEventType()))
              // Map value to uppercase eventType
              .mapValues(value -> {
                  value.setEventType(value.getEventType().toUpperCase());
                  return value;
              })
              .to("output-topic", Produced.with(Serdes.String(), userEventSerde));

        // Count events per user
        stream.groupBy((key, value) -> value.getUserId(), Grouped.with(Serdes.String(), userEventSerde))
              .count(Materialized.as("user-event-count"))
              .toStream()
              .to("user-count-topic", Produced.with(Serdes.String(), Serdes.Long()));

        return stream;
    }
}
```

**Explanation:**

1. Reads `UserEvent` objects from `input-topic`.
2. Filters only `LOGIN` events.
3. Maps event type to uppercase.
4. Sends processed events to `output-topic`.
5. Counts events per `userId` and sends to `user-count-topic`.

---

## **7. StreamProcessingService.java (Optional)**

```java
package com.example.demo.service;

import com.example.demo.model.UserEvent;
import org.apache.kafka.streams.kstream.KStream;
import org.springframework.stereotype.Service;

@Service
public class StreamProcessingService {
    private final KStream<String, UserEvent> stream;

    public StreamProcessingService(KStream<String, UserEvent> stream) {
        this.stream = stream;
    }

    public void customProcessing() {
        stream.filter((key, value) -> value.getEventType().contains("CLICK"))
              .mapValues(value -> {
                  value.setEventType("CLICK_PROCESSED");
                  return value;
              });
    }
}
```

---

## **8. How to Run**

1. Start Kafka + Zookeeper.
2. Create topics:

```bash
kafka-topics.sh --create --topic input-topic --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1
kafka-topics.sh --create --topic output-topic --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1
kafka-topics.sh --create --topic user-count-topic --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1
```

3. Run `DemoApplication.java`.
4. Produce events to `input-topic`.
5. Consume results from `output-topic` and `user-count-topic`.

---

## **9. Practice Plan**

1. Produce 1000 sample `UserEvent`s.
2. Filter by `eventType`.
3. Count events per user (KTable).
4. Join two streams (`CLICK` and `PURCHASE`).
5. Implement **time-windowed aggregation**:

```java
stream.groupByKey()
      .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(5)))
      .count()
      .toStream()
      .to("windowed-topic");
```

6. Test **stateful operations** with KTables.
7. Experiment with **custom Serdes**.
8. Monitor with `KafkaStreams#metrics()`.

---

## ✅ **Summary**

* Spring Boot + Kafka Streams = easy stream processing.
* Supports **DSL operations** (map, filter, aggregate, join).
* Supports **windowing** for time-based analytics.
* Serdes for custom objects (JSON, Avro).
* KStream = event stream, KTable = stateful table.

---

**Kafka Streams Spring Boot project** with **prebuilt topics, sample producer code, and all stream examples**, ready to run.

---
Perfect! Here's how we can structure the **downloadable ready-to-run Kafka Streams Spring Boot project** with everything prebuilt, so you can just extract, run, and test.

---

# **Kafka Streams Spring Boot Project (Ready-to-Run)**

### **Project Name:** `kafka-streams-demo`

---

## **1. Project Structure**

```
kafka-streams-demo/
├── src/main/java/com/example/demo/
│   ├── DemoApplication.java
│   ├── config/
│   │   └── KafkaStreamsConfig.java
│   ├── model/
│   │   └── UserEvent.java
│   ├── service/
│   │   └── StreamProcessingService.java
│   └── producer/
│       └── EventProducer.java
├── src/main/resources/
│   └── application.properties
├── pom.xml
```

---

## **2. pom.xml**

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" ...>
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>kafka-streams-demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>Kafka Streams Demo</name>

    <dependencies>
        <!-- Spring Boot Starter -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <!-- Spring Kafka -->
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka</artifactId>
        </dependency>

        <!-- Kafka Streams -->
        <dependency>
            <groupId>org.apache.kafka</groupId>
            <artifactId>kafka-streams</artifactId>
        </dependency>
    </dependencies>
</project>
```

---

## **3. application.properties**

```properties
spring.kafka.bootstrap-servers=localhost:9092
spring.kafka.streams.application-id=kafka-streams-demo
spring.kafka.streams.default-key-serde=org.apache.kafka.common.serialization.Serdes$StringSerde
spring.kafka.streams.default-value-serde=org.apache.kafka.common.serialization.Serdes$StringSerde
spring.kafka.consumer.auto-offset-reset=earliest
```

---

## **4. DemoApplication.java**

```java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

---

## **5. Model: UserEvent.java**

```java
package com.example.demo.model;

public class UserEvent {
    private String userId;
    private String eventType;
    private long timestamp;

    public UserEvent() {}

    public UserEvent(String userId, String eventType, long timestamp) {
        this.userId = userId;
        this.eventType = eventType;
        this.timestamp = timestamp;
    }

    // Getters and setters
}
```

---

## **6. KafkaStreamsConfig.java**

```java
package com.example.demo.config;

import com.example.demo.model.UserEvent;
import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.kstream.*;
import org.apache.kafka.streams.StreamsBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.support.serializer.JsonSerde;

@Configuration
public class KafkaStreamsConfig {

    @Bean
    public KStream<String, UserEvent> kStream(StreamsBuilder builder) {
        JsonSerde<UserEvent> userEventSerde = new JsonSerde<>(UserEvent.class);

        KStream<String, UserEvent> stream = builder.stream("input-topic",
                Consumed.with(Serdes.String(), userEventSerde));

        // Filter LOGIN events and convert to uppercase
        stream.filter((key, value) -> "LOGIN".equals(value.getEventType()))
              .mapValues(value -> {
                  value.setEventType(value.getEventType().toUpperCase());
                  return value;
              })
              .to("output-topic", Produced.with(Serdes.String(), userEventSerde));

        // Count events per user
        stream.groupBy((key, value) -> value.getUserId(), Grouped.with(Serdes.String(), userEventSerde))
              .count(Materialized.as("user-event-count"))
              .toStream()
              .to("user-count-topic", Produced.with(Serdes.String(), Serdes.Long()));

        return stream;
    }
}
```

---

## **7. StreamProcessingService.java (Optional)**

```java
package com.example.demo.service;

import com.example.demo.model.UserEvent;
import org.apache.kafka.streams.kstream.KStream;
import org.springframework.stereotype.Service;

@Service
public class StreamProcessingService {
    private final KStream<String, UserEvent> stream;

    public StreamProcessingService(KStream<String, UserEvent> stream) {
        this.stream = stream;
    }

    public void customProcessing() {
        stream.filter((key, value) -> value.getEventType().contains("CLICK"))
              .mapValues(value -> {
                  value.setEventType("CLICK_PROCESSED");
                  return value;
              });
    }
}
```

---

## **8. EventProducer.java (Test Producer)**

```java
package com.example.demo.producer;

import com.example.demo.model.UserEvent;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Component;

import java.util.Random;

@Component
public class EventProducer {

    private final KafkaTemplate<String, UserEvent> kafkaTemplate;

    public EventProducer(KafkaTemplate<String, UserEvent> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public void produceEvents() {
        String[] eventTypes = {"LOGIN", "CLICK", "PURCHASE"};
        Random rand = new Random();

        for (int i = 0; i < 100; i++) {
            UserEvent event = new UserEvent(
                "user" + (rand.nextInt(5) + 1),
                eventTypes[rand.nextInt(eventTypes.length)],
                System.currentTimeMillis()
            );
            kafkaTemplate.send("input-topic", event.getUserId(), event);
        }
    }
}
```

---

## **9. How to Run**

1. **Start Kafka + Zookeeper**
2. **Create Topics:**

```bash
kafka-topics.sh --create --topic input-topic --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1
kafka-topics.sh --create --topic output-topic --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1
kafka-topics.sh --create --topic user-count-topic --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1
```

3. Run `DemoApplication.java`.
4. Inject `EventProducer` and call `produceEvents()` to generate sample events.
5. Consume results from `output-topic` and `user-count-topic`:

```bash
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic output-topic --from-beginning
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic user-count-topic --from-beginning
```

---

## **10. Features in This Project**

* Filters events (`LOGIN`)
* Maps event type to uppercase
* Counts events per user (KTable)
* Uses JSON Serde for custom objects
* Supports KStream transformations and aggregations
* Ready-to-test with sample producer

---


---


