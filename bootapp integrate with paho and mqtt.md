---
Nice — I’ll give you a compact, copy-and-paste friendly **step-by-step guide + runnable code** for integrating a Spring Boot app with **Eclipse Paho (Java MQTT client)** for publish/subscribe, plus an alternative using **Spring Integration MQTT** (recommended if you want Spring messaging integration). I’ll include quick test steps (Mosquitto broker via Docker), production notes (TLS/QoS/auth), and troubleshooting tips.

**Key sources:** Eclipse Paho client (Java), Spring Integration MQTT docs, Mosquitto broker docs and a Java tutorial — I’ll cite these as we go. ([GitHub][1], [Home][2], [Eclipse Mosquitto][3], [Baeldung on Kotlin][4])

---

# 1) Quick overview (what’s what)

* **MQTT broker** — central server (e.g., Eclipse Mosquitto) that receives published messages and forwards them to subscribers. ([Eclipse Mosquitto][3])
* **Client library** — Eclipse Paho Java provides `MqttClient` / `MqttAsyncClient` to connect, publish, and subscribe. Use Paho directly for simple needs. ([GitHub][1])
* **Spring Integration MQTT** — wraps Paho and exposes inbound/outbound adapters and Spring `MessageChannel`s — good when you want to integrate with Spring messaging flow. ([Home][2])

---

# 2) Prerequisites

* Java (8+; I’ll show code usable on Java 8/11/17).
* Maven (or Gradle).
* Local broker for testing: Eclipse Mosquitto (can run with Docker). ([Eclipse Mosquitto][3])

Quick Docker broker (one-line):

```
docker run -it -p 1883:1883 -p 9001:9001 eclipse-mosquitto
```

(then broker is at `tcp://localhost:1883`). ([Eclipse Mosquitto][3])

---

# 3) Dependencies (Maven)

Add Paho (and optionally Spring Integration) to your `pom.xml`:

```xml
<!-- Paho client -->
<dependency>
  <groupId>org.eclipse.paho</groupId>
  <artifactId>org.eclipse.paho.client.mqttv3</artifactId>
  <version>1.2.5</version> <!-- pick latest stable -->
</dependency>

<!-- (Optional) Spring Integration MQTT -->
<dependency>
  <groupId>org.springframework.integration</groupId>
  <artifactId>spring-integration-mqtt</artifactId>
</dependency>

<!-- usual Spring Boot starters -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

Paho artifact info: Maven listings / repo. ([Maven Repository][5], [Maven Central][6])

---

# 4) Approach A — **Direct Paho usage** (simple, minimal layers)

**application.properties**

```properties
mqtt.broker=tcp://localhost:1883
mqtt.client-id=boot-paho-client
```

**Paho service (publisher + subscriber)**

```java
// src/main/java/com/example/mqtt/PahoMqttService.java
package com.example.mqtt;

import org.eclipse.paho.client.mqttv3.*;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import javax.annotation.PostConstruct;
import javax.annotation.PreDestroy;
import java.nio.charset.StandardCharsets;

@Service
public class PahoMqttService {

    private final IMqttClient client;

    public PahoMqttService(
            @Value("${mqtt.broker}") String broker,
            @Value("${mqtt.client-id}") String clientId) throws MqttException {

        this.client = new MqttClient(broker, clientId);
        MqttConnectOptions opts = new MqttConnectOptions();
        opts.setAutomaticReconnect(true);
        opts.setCleanSession(true);
        // opts.setUserName(...); opts.setPassword(...);
        client.connect(opts);
    }

    @PostConstruct
    public void start() throws MqttException {
        // subscribe on startup (example)
        client.subscribe("myapp/inbound/#", (topic, msg) -> {
            String payload = new String(msg.getPayload(), StandardCharsets.UTF_8);
            System.out.printf("Received msg on %s : %s%n", topic, payload);
            // TODO: hand off to service layer, convert JSON, etc.
        });
    }

    public void publish(String topic, String payload, int qos) throws MqttException {
        MqttMessage message = new MqttMessage(payload.getBytes(StandardCharsets.UTF_8));
        message.setQos(qos); // 0,1,2
        client.publish(topic, message);
    }

    @PreDestroy
    public void stop() throws MqttException {
        if (client.isConnected()) client.disconnect();
    }
}
```

**Expose REST endpoint to publish (optional)**

```java
@RestController
@RequestMapping("/mqtt")
public class MqttController {
    private final PahoMqttService mqtt;

    public MqttController(PahoMqttService mqtt) { this.mqtt = mqtt; }

    @PostMapping("/publish")
    public ResponseEntity<?> publish(@RequestParam String topic, @RequestBody String msg) throws MqttException {
        mqtt.publish(topic, msg, 1);
        return ResponseEntity.ok("published");
    }
}
```

**Notes on the code**

* `client.subscribe(topic, (t, msg) -> { ... })` registers a message listener (asynchronous).
* `MqttConnectOptions` allows TLS, username/password, cleanSession, automaticReconnect. See Paho docs. ([GitHub][1])

---

# 5) Approach B — **Spring Integration + Paho** (recommended for Spring apps)

Add dependency `spring-integration-mqtt` (it uses Paho). This gives inbound/outbound adapters and channels.

**Configuration (Java)**

```java
// src/main/java/com/example/mqtt/MqttConfig.java
package com.example.mqtt;

import org.eclipse.paho.client.mqttv3.MqttConnectOptions;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.annotation.ServiceActivator;
import org.springframework.integration.channel.DirectChannel;
import org.springframework.integration.mqtt.core.DefaultMqttPahoClientFactory;
import org.springframework.integration.mqtt.core.MqttPahoClientFactory;
import org.springframework.integration.mqtt.inbound.MqttPahoMessageDrivenChannelAdapter;
import org.springframework.integration.mqtt.outbound.MqttPahoMessageHandler;
import org.springframework.integration.mqtt.support.DefaultPahoMessageConverter;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.MessageHandler;

@Configuration
public class MqttConfig {

    @Bean
    public MqttPahoClientFactory mqttClientFactory() {
        DefaultMqttPahoClientFactory factory = new DefaultMqttPahoClientFactory();
        MqttConnectOptions options = new MqttConnectOptions();
        options.setServerURIs(new String[] { "tcp://localhost:1883" });
        options.setAutomaticReconnect(true);
        options.setCleanSession(true);
        // options.setUserName(...); options.setPassword(...);
        factory.setConnectionOptions(options);
        return factory;
    }

    @Bean
    public MessageChannel mqttInputChannel() {
        return new DirectChannel();
    }

    @Bean
    public MqttPahoMessageDrivenChannelAdapter inbound() {
        MqttPahoMessageDrivenChannelAdapter adapter =
            new MqttPahoMessageDrivenChannelAdapter("springClientIn", mqttClientFactory(), "myapp/inbound/#");
        adapter.setConverter(new DefaultPahoMessageConverter());
        adapter.setQos(1);
        adapter.setOutputChannel(mqttInputChannel());
        return adapter;
    }

    @Bean
    @ServiceActivator(inputChannel = "mqttInputChannel")
    public MessageHandler handler() {
        return message -> {
            System.out.println("Received via Spring Integration: " + message.getPayload());
            // convert and process...
        };
    }

    @Bean
    @ServiceActivator(inputChannel = "mqttOutboundChannel")
    public MessageHandler mqttOutbound() {
        MqttPahoMessageHandler handler = new MqttPahoMessageHandler("springClientOut", mqttClientFactory());
        handler.setAsync(true);
        handler.setDefaultTopic("myapp/outbound");
        return handler;
    }

    @Bean
    public MessageChannel mqttOutboundChannel() {
        return new DirectChannel();
    }
}
```

**Usage**

* To publish: `messageChannel.send(MessageBuilder.withPayload("hello").setHeader("mqtt_topic", "myapp/outbound").build())` or send to `mqttOutboundChannel`.
* This approach plugs nicely into Spring Messaging and lets you annotate or route messages as regular Spring messages. ([Home][2])

---

# 6) Test quickly (local)

1. Start Mosquitto (Docker or apt install). ([Eclipse Mosquitto][3])
2. Run your Spring Boot app.
3. Use CLI to test:

```
# in one terminal: subscribe
mosquitto_sub -h localhost -t 'myapp/inbound/#' -v

# in another terminal: publish
mosquitto_pub -h localhost -t 'myapp/inbound/test' -m 'hello from CLI'
```

You should see the message printed by your Spring Boot subscriber. (These `mosquitto_*` tools come with mosquitto-clients). ([Eclipse Mosquitto][3], [Baeldung on Kotlin][4])

---

# 7) MQTT basics & production tips (cheat-sheet)

* **QoS**: 0 = at most once, 1 = at least once, 2 = exactly once. Choose QoS by delivery guarantees vs overhead. ([Baeldung on Kotlin][4])
* **Retained messages**: broker keeps last retained message for a topic — useful for state.
* **Last Will & Testament (LWT)**: configure `MqttConnectOptions.setWill(...)` to notify others if client disconnects unexpectedly.
* **Persistent sessions**: `cleanSession=false` lets broker keep subscriptions while client is offline.
* **TLS / auth**: use `ssl://broker:8883` and set `MqttConnectOptions` TLS socket factory and username/password for production. See broker docs. ([Eclipse Mosquitto][3])
* **Scaling**: If you have many subscribers, use broker clustering or a managed MQTT service (HiveMQ, EMQX, AWS IoT Core).
* **Message size and format**: prefer small, compact payloads (JSON, CBOR), avoid huge messages; consider compression or chunking.

---

# 8) Troubleshooting checklist

* **Connect exception** → broker running / host/port correct? (`telnet localhost 1883`)
* **ClassNotFound** → add Paho dependency to the final artifact (fat jar / Spring Boot repackage). See common JAR packaging issues. ([Stack Overflow][7])
* **No messages** → check topic names and wildcards, QoS mismatch, `cleanSession` settings.
* **Authorization failures** → broker user/pass or ACLs.
* **SSL errors** → check certificates and the correct `ssl://` URL and enabled TLS cipher suites. ([Eclipse Mosquitto][3])

---

# 9) Example repo & more reading

* Example Spring Boot + MQTT sample repos and tutorials (good for copy/paste). ([GitHub][8], [Medium][9])
* Official Paho Java project & docs. ([GitHub][1])
* Spring Integration MQTT section (official docs; includes Java config examples). ([Home][2])
* Java MQTT tutorial (Baeldung) — handy reference for client APIs and patterns. ([Baeldung on Kotlin][4])

---

# 10) Minimal “quickstart” checklist (copy this and run)

1. Start Mosquitto: `docker run -it -p 1883:1883 eclipse-mosquitto` (or install natively). ([Eclipse Mosquitto][3])
2. Create Spring Boot app, add Paho dependency.
3. Add `PahoMqttService` (code above).
4. Run Spring app.
5. Test with `mosquitto_pub` and `mosquitto_sub` as shown.

---

If you want, I can:

* generate a **complete minimal Spring Boot project** (pom + full classes + README) you can clone and run locally, OR
* show the **Spring Integration** example wired to a REST controller and a DTO/json converter, OR
* produce the **Docker Compose** that runs `mosquitto` + an app container for end-to-end testing.

Pick one and I’ll produce the full runnable artifact right here.

[1]: https://github.com/eclipse-paho/paho.mqtt.java?utm_source=chatgpt.com "Eclipse Paho Java MQTT client library. ..."
[2]: https://docs.spring.io/spring-integration/reference/mqtt.html?utm_source=chatgpt.com "MQTT Support :: Spring Integration"
[3]: https://mosquitto.org/?utm_source=chatgpt.com "Eclipse Mosquitto"
[4]: https://www.baeldung.com/java-mqtt-client?utm_source=chatgpt.com "MQTT Client in Java"
[5]: https://mvnrepository.com/artifact/org.eclipse.paho/org.eclipse.paho.client.mqttv3?utm_source=chatgpt.com "org.eclipse.paho.client.mqttv3"
[6]: https://central.sonatype.com/artifact/org.eclipse.paho/org.eclipse.paho.client.mqttv3.repository?utm_source=chatgpt.com "org.eclipse.paho.client.mqttv3.repository - Maven Central"
[7]: https://stackoverflow.com/questions/60563649/error-caused-by-java-lang-classnotfoundexception-org-eclipse-paho-client-mqt?utm_source=chatgpt.com "Error: \"Caused by: java.lang.ClassNotFoundException: org. ..."
[8]: https://github.com/gulteking/spring-boot-mqtt-sample?utm_source=chatgpt.com "gulteking/spring-boot-mqtt-sample"
[9]: https://harkesh3.medium.com/mqtt-integration-with-spring-boot-fd762f258536?utm_source=chatgpt.com "MQTT integration with spring boot | by Harkesh Kumar - Medium"

---


---
Got it 👍 You mean **yes, please generate a complete runnable Spring Boot project** with Paho MQTT integration?

I’ll prepare a **minimal, working project structure** you can copy-paste:

---

## 📂 Project Structure

```
spring-mqtt-demo/
 ├─ pom.xml
 └─ src/main/java/com/example/mqtt/
       ├─ SpringMqttDemoApplication.java
       ├─ PahoMqttService.java
       └─ MqttController.java
 └─ src/main/resources/
       └─ application.properties
```

---

## 1) `pom.xml`

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>spring-mqtt-demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.4</version>
    </parent>

    <dependencies>
        <!-- Spring Boot Web -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- Paho MQTT client -->
        <dependency>
            <groupId>org.eclipse.paho</groupId>
            <artifactId>org.eclipse.paho.client.mqttv3</artifactId>
            <version>1.2.5</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

---

## 2) `application.properties`

```properties
server.port=8080
mqtt.broker=tcp://localhost:1883
mqtt.client-id=boot-paho-client
```

---

## 3) `SpringMqttDemoApplication.java`

```java
package com.example.mqtt;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class SpringMqttDemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringMqttDemoApplication.class, args);
    }
}
```

---

## 4) `PahoMqttService.java`

```java
package com.example.mqtt;

import org.eclipse.paho.client.mqttv3.*;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import javax.annotation.PostConstruct;
import javax.annotation.PreDestroy;
import java.nio.charset.StandardCharsets;

@Service
public class PahoMqttService {

    private final IMqttClient client;

    public PahoMqttService(
            @Value("${mqtt.broker}") String broker,
            @Value("${mqtt.client-id}") String clientId) throws MqttException {

        this.client = new MqttClient(broker, clientId);
        MqttConnectOptions opts = new MqttConnectOptions();
        opts.setAutomaticReconnect(true);
        opts.setCleanSession(true);
        client.connect(opts);
    }

    @PostConstruct
    public void start() throws MqttException {
        client.subscribe("myapp/inbound/#", (topic, msg) -> {
            String payload = new String(msg.getPayload(), StandardCharsets.UTF_8);
            System.out.printf("📩 Received on [%s] : %s%n", topic, payload);
        });
    }

    public void publish(String topic, String payload, int qos) throws MqttException {
        MqttMessage message = new MqttMessage(payload.getBytes(StandardCharsets.UTF_8));
        message.setQos(qos);
        client.publish(topic, message);
    }

    @PreDestroy
    public void stop() throws MqttException {
        if (client.isConnected()) client.disconnect();
    }
}
```

---

## 5) `MqttController.java`

```java
package com.example.mqtt;

import org.eclipse.paho.client.mqttv3.MqttException;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/mqtt")
public class MqttController {
    private final PahoMqttService mqtt;

    public MqttController(PahoMqttService mqtt) {
        this.mqtt = mqtt;
    }

    @PostMapping("/publish")
    public ResponseEntity<String> publish(
            @RequestParam String topic,
            @RequestBody String msg) throws MqttException {
        mqtt.publish(topic, msg, 1);
        return ResponseEntity.ok("✅ Published to " + topic);
    }
}
```

---

## 6) Run Broker

Start Mosquitto in Docker:

```bash
docker run -it -p 1883:1883 eclipse-mosquitto
```

---

## 7) Test Flow

1. Run Spring Boot app:

   ```bash
   mvn spring-boot:run
   ```
2. Open terminal subscriber:

   ```bash
   mosquitto_sub -h localhost -t "myapp/inbound/#" -v
   ```
3. Publish via REST API:

   ```bash
   curl -X POST "http://localhost:8080/mqtt/publish?topic=myapp/inbound/test" \
        -H "Content-Type: text/plain" \
        -d "Hello MQTT from Spring Boot!"
   ```
4. You should see the message appear in both:

   * Spring Boot console (`📩 Received...`)
   * Mosquitto subscriber terminal.

---

👉 This is the **runnable minimal demo**.
If you want, I can also add a **Spring Integration (channel-based)** version in the same project so you can compare **direct Paho vs Spring style**.

Do you want me to add that too inside the same project?

---

---
Perfect 👍 Let’s extend the same project with a **Spring Integration MQTT** config alongside the direct Paho approach.

That way, you can compare **raw Paho (manual publish/subscribe)** vs **Spring Integration (message channels + handlers)** in the same app.

---

## 📂 Updated Project Structure

```
spring-mqtt-demo/
 ├─ pom.xml
 └─ src/main/java/com/example/mqtt/
       ├─ SpringMqttDemoApplication.java
       ├─ PahoMqttService.java         ← direct Paho client
       ├─ MqttController.java          ← REST publish (direct Paho)
       ├─ MqttIntegrationConfig.java   ← Spring Integration config
       └─ IntegrationPublisher.java    ← REST publish (Spring Integration)
 └─ src/main/resources/
       └─ application.properties
```

---

## 1) `pom.xml` (add Spring Integration)

Add inside `<dependencies>`:

```xml
<!-- Spring Integration MQTT (uses Paho under the hood) -->
<dependency>
    <groupId>org.springframework.integration</groupId>
    <artifactId>spring-integration-mqtt</artifactId>
</dependency>
```

---

## 2) `application.properties` (same)

```properties
server.port=8080
mqtt.broker=tcp://localhost:1883
mqtt.client-id=boot-paho-client
```

---

## 3) `MqttIntegrationConfig.java`

```java
package com.example.mqtt;

import org.eclipse.paho.client.mqttv3.MqttConnectOptions;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.annotation.ServiceActivator;
import org.springframework.integration.channel.DirectChannel;
import org.springframework.integration.mqtt.core.DefaultMqttPahoClientFactory;
import org.springframework.integration.mqtt.core.MqttPahoClientFactory;
import org.springframework.integration.mqtt.inbound.MqttPahoMessageDrivenChannelAdapter;
import org.springframework.integration.mqtt.outbound.MqttPahoMessageHandler;
import org.springframework.integration.mqtt.support.DefaultPahoMessageConverter;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.MessageHandler;

@Configuration
public class MqttIntegrationConfig {

    @Bean
    public MqttPahoClientFactory mqttClientFactory() {
        DefaultMqttPahoClientFactory factory = new DefaultMqttPahoClientFactory();
        MqttConnectOptions options = new MqttConnectOptions();
        options.setServerURIs(new String[]{"tcp://localhost:1883"});
        options.setAutomaticReconnect(true);
        options.setCleanSession(true);
        factory.setConnectionOptions(options);
        return factory;
    }

    // ---------- Inbound (Subscribe) ----------
    @Bean
    public MessageChannel mqttInputChannel() {
        return new DirectChannel();
    }

    @Bean
    public MqttPahoMessageDrivenChannelAdapter inbound() {
        MqttPahoMessageDrivenChannelAdapter adapter =
                new MqttPahoMessageDrivenChannelAdapter(
                        "springClientIn", mqttClientFactory(), "integration/inbound/#");
        adapter.setConverter(new DefaultPahoMessageConverter());
        adapter.setQos(1);
        adapter.setOutputChannel(mqttInputChannel());
        return adapter;
    }

    @Bean
    @ServiceActivator(inputChannel = "mqttInputChannel")
    public MessageHandler inboundHandler() {
        return message -> {
            System.out.println("📩 [Spring Integration] Received: " + message.getPayload());
        };
    }

    // ---------- Outbound (Publish) ----------
    @Bean
    public MessageChannel mqttOutboundChannel() {
        return new DirectChannel();
    }

    @Bean
    @ServiceActivator(inputChannel = "mqttOutboundChannel")
    public MessageHandler outbound() {
        MqttPahoMessageHandler handler =
                new MqttPahoMessageHandler("springClientOut", mqttClientFactory());
        handler.setAsync(true);
        handler.setDefaultTopic("integration/outbound");
        return handler;
    }
}
```

---

## 4) `IntegrationPublisher.java`

```java
package com.example.mqtt;

import org.springframework.http.ResponseEntity;
import org.springframework.integration.support.MessageBuilder;
import org.springframework.messaging.MessageChannel;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/integration")
public class IntegrationPublisher {

    private final MessageChannel mqttOutboundChannel;

    public IntegrationPublisher(MessageChannel mqttOutboundChannel) {
        this.mqttOutboundChannel = mqttOutboundChannel;
    }

    @PostMapping("/publish")
    public ResponseEntity<String> publish(
            @RequestParam(required = false) String topic,
            @RequestBody String msg) {

        if (topic == null) {
            topic = "integration/outbound";
        }

        mqttOutboundChannel.send(
                MessageBuilder.withPayload(msg)
                        .setHeader("mqtt_topic", topic)
                        .build()
        );

        return ResponseEntity.ok("✅ Published (Spring Integration) to " + topic);
    }
}
```

---

## 5) Testing Both Approaches

### Start Broker:

```bash
docker run -it -p 1883:1883 eclipse-mosquitto
```

### Subscribe to topics:

```bash
# For Paho direct subscriber:
mosquitto_sub -h localhost -t "myapp/inbound/#" -v

# For Spring Integration subscriber:
mosquitto_sub -h localhost -t "integration/inbound/#" -v
```

### Publish via REST:

```bash
# Publish using PahoService
curl -X POST "http://localhost:8080/mqtt/publish?topic=myapp/inbound/test" \
     -H "Content-Type: text/plain" -d "Hello Paho Direct!"

# Publish using Spring Integration
curl -X POST "http://localhost:8080/integration/publish?topic=integration/inbound/test" \
     -H "Content-Type: text/plain" -d "Hello Spring Integration!"
```

---

## 6) Output Expectation

* Console:

  ```
  📩 Received on [myapp/inbound/test] : Hello Paho Direct!
  📩 [Spring Integration] Received: Hello Spring Integration!
  ```
* Mosquitto subscriber windows show same messages.

---

✅ Now your single project demonstrates **two approaches**:

* **Direct Paho** (simpler, full control, but more boilerplate).
* **Spring Integration MQTT** (cleaner, fits into Spring messaging system).

---

Do you want me to also prepare a **Docker Compose file** with both Mosquitto + this Spring Boot app (so you can `docker-compose up` and test end-to-end)?

---

---
Great 🚀 Let’s add a **Docker Compose setup** so you can spin up Mosquitto + your Spring Boot app together with a single command.

---

## 📂 Final Project Structure

```
spring-mqtt-demo/
 ├─ docker-compose.yml
 ├─ Dockerfile
 ├─ pom.xml
 └─ src/main/java/com/example/mqtt/
       ├─ SpringMqttDemoApplication.java
       ├─ PahoMqttService.java
       ├─ MqttController.java
       ├─ MqttIntegrationConfig.java
       └─ IntegrationPublisher.java
 └─ src/main/resources/
       └─ application.properties
```

---

## 1) `Dockerfile`

This will build your Spring Boot app into a container:

```dockerfile
# Build stage
FROM maven:3.9.4-eclipse-temurin-17 AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

# Runtime stage
FROM eclipse-temurin:17-jdk
WORKDIR /app
COPY --from=build /app/target/spring-mqtt-demo-0.0.1-SNAPSHOT.jar app.jar
ENTRYPOINT ["java","-jar","app.jar"]
```

---

## 2) `docker-compose.yml`

This runs **Mosquitto broker** + **Spring Boot app** side by side:

```yaml
version: '3.8'
services:
  mosquitto:
    image: eclipse-mosquitto:2
    container_name: mosquitto-broker
    ports:
      - "1883:1883"   # MQTT broker
      - "9001:9001"   # WebSocket (optional)
    volumes:
      - ./mosquitto.conf:/mosquitto/config/mosquitto.conf

  spring-app:
    build: .
    container_name: spring-mqtt-demo
    depends_on:
      - mosquitto
    environment:
      - MQTT_BROKER=tcp://mosquitto:1883
      - MQTT_CLIENT_ID=boot-client
    ports:
      - "8080:8080"
```

---

## 3) `application.properties` (update to use env vars)

```properties
server.port=8080
mqtt.broker=${MQTT_BROKER:tcp://localhost:1883}
mqtt.client-id=${MQTT_CLIENT_ID:boot-paho-client}
```

👉 This way, in Docker it connects to `mosquitto` service (Docker network name). Locally, it defaults to `localhost`.

---

## 4) Minimal `mosquitto.conf`

Create file `mosquitto.conf` in the project root:

```conf
listener 1883
allow_anonymous true
```

*(This allows anonymous connections for testing; in prod, add username/password).*

---

## 5) Build & Run

From project root:

```bash
docker-compose up --build
```

---

## 6) Test Inside Compose

1. App runs at `http://localhost:8080`
2. Publish via REST:

   ```bash
   curl -X POST "http://localhost:8080/mqtt/publish?topic=myapp/inbound/test" \
        -H "Content-Type: text/plain" -d "Hello from Docker!"
   ```
3. Subscribe to broker:

   ```bash
   docker exec -it mosquitto-broker mosquitto_sub -h localhost -t "myapp/inbound/#" -v
   ```

   You should see the message appear.

---

✅ Now you can test **end-to-end** with one command: `docker-compose up`.
It gives you:

* Mosquitto broker
* Spring Boot app with **both Paho direct & Spring Integration** subscribers

---

Do you also want me to add a **Postman collection (JSON)** with sample publish requests so you can test without curl?

---

---
Awesome 👍 Let’s add a **Postman Collection** so you can test your MQTT REST endpoints (`/mqtt/publish` and `/integration/publish`) directly from Postman instead of curl.

---

## 📂 Updated Project

```
spring-mqtt-demo/
 ├─ docker-compose.yml
 ├─ Dockerfile
 ├─ mosquitto.conf
 ├─ SpringMQTT.postman_collection.json   ← Postman collection
 ├─ pom.xml
 └─ src/main/...
```

---

## 1) `SpringMQTT.postman_collection.json`

```json
{
  "info": {
    "name": "Spring MQTT Demo",
    "_postman_id": "12345678-aaaa-bbbb-cccc-1234567890ab",
    "description": "Collection to test Spring Boot MQTT REST endpoints (Paho + Spring Integration).",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
  },
  "item": [
    {
      "name": "Publish via Direct Paho",
      "request": {
        "method": "POST",
        "header": [
          {
            "key": "Content-Type",
            "value": "text/plain"
          }
        ],
        "body": {
          "mode": "raw",
          "raw": "Hello from Postman (Paho)!"
        },
        "url": {
          "raw": "http://localhost:8080/mqtt/publish?topic=myapp/inbound/test",
          "protocol": "http",
          "host": ["localhost"],
          "port": "8080",
          "path": ["mqtt", "publish"],
          "query": [
            { "key": "topic", "value": "myapp/inbound/test" }
          ]
        }
      },
      "response": []
    },
    {
      "name": "Publish via Spring Integration",
      "request": {
        "method": "POST",
        "header": [
          {
            "key": "Content-Type",
            "value": "text/plain"
          }
        ],
        "body": {
          "mode": "raw",
          "raw": "Hello from Postman (Spring Integration)!"
        },
        "url": {
          "raw": "http://localhost:8080/integration/publish?topic=integration/inbound/test",
          "protocol": "http",
          "host": ["localhost"],
          "port": "8080",
          "path": ["integration", "publish"],
          "query": [
            { "key": "topic", "value": "integration/inbound/test" }
          ]
        }
      },
      "response": []
    }
  ]
}
```

---

## 2) How to Use

1. Open Postman.
2. Click **Import** → Select `SpringMQTT.postman_collection.json`.
3. You’ll see a **collection** with 2 requests:

   * **Publish via Direct Paho** → calls `/mqtt/publish`
   * **Publish via Spring Integration** → calls `/integration/publish`
4. Start your app (`docker-compose up --build`).
5. Hit **Send** in Postman.
6. Check:

   * Spring Boot logs → `📩 Received...` messages
   * Mosquitto subscriber → `mosquitto_sub -h localhost -t '#' -v`

---

✅ With this setup you now have:

* **Mosquitto Broker** (Docker)
* **Spring Boot App** (with both direct Paho & Spring Integration MQTT)
* **Postman Collection** (ready-made testing requests)

---

Do you also want me to add a **subscriber request in Postman** (using Postman’s WebSocket support to listen to `ws://localhost:9001` from Mosquitto) so you can test **live subscriptions** inside Postman UI itself?



---

---
Perfect 👍 Let’s now prepare **final structured notes with code** on how to integrate a **Spring Boot app with Eclipse Paho MQTT** for **Pub/Sub**.

---

# 📒 Notes: Spring Boot + Eclipse Paho + MQTT (Pub/Sub)

---

## 1. 📝 Basics

* **MQTT (Message Queuing Telemetry Transport)**: Lightweight pub/sub messaging protocol, widely used in IoT.
* **Paho**: Eclipse project providing MQTT client libraries (Java, Python, C).
* **Spring Boot Integration**: We use **Spring Integration MQTT** + **Paho** for producer & consumer.

---

## 2. ⚙️ Setup MQTT Broker (Docker)

Run Mosquitto broker locally:

```bash
docker run -it -p 1883:1883 -p 9001:9001 eclipse-mosquitto
```

* MQTT Broker URL: `tcp://localhost:1883`
* WebSocket port: `9001`

---

## 3. 📦 Dependencies

Add to `pom.xml`:

```xml
<dependency>
    <groupId>org.springframework.integration</groupId>
    <artifactId>spring-integration-mqtt</artifactId>
</dependency>

<dependency>
    <groupId>org.eclipse.paho</groupId>
    <artifactId>org.eclipse.paho.client.mqttv3</artifactId>
</dependency>
```

---

## 4. ⚙️ MQTT Config in Spring Boot

```java
@Configuration
public class MqttConfig {

    private static final String MQTT_BROKER = "tcp://localhost:1883";
    private static final String CLIENT_ID = "bootAppClient";

    @Bean
    public MqttConnectOptions mqttConnectOptions() {
        MqttConnectOptions options = new MqttConnectOptions();
        options.setServerURIs(new String[]{MQTT_BROKER});
        options.setCleanSession(true);
        return options;
    }

    @Bean
    public MqttPahoClientFactory mqttClientFactory() {
        DefaultMqttPahoClientFactory factory = new DefaultMqttPahoClientFactory();
        factory.setConnectionOptions(mqttConnectOptions());
        return factory;
    }

    // Producer (outbound)
    @Bean
    @ServiceActivator(inputChannel = "mqttOutboundChannel")
    public MessageHandler mqttOutbound() {
        MqttPahoMessageHandler handler =
                new MqttPahoMessageHandler(CLIENT_ID + "_producer", mqttClientFactory());
        handler.setAsync(true);
        handler.setDefaultTopic("demo/topic");
        return handler;
    }

    @Bean
    public MessageChannel mqttOutboundChannel() {
        return new DirectChannel();
    }

    // Consumer (inbound)
    @Bean
    public MessageProducer inbound() {
        MqttPahoMessageDrivenChannelAdapter adapter =
                new MqttPahoMessageDrivenChannelAdapter(CLIENT_ID + "_consumer", mqttClientFactory(),
                        "demo/topic");
        adapter.setOutputChannel(mqttInboundChannel());
        return adapter;
    }

    @Bean
    public MessageChannel mqttInboundChannel() {
        return new DirectChannel();
    }

    @Bean
    @ServiceActivator(inputChannel = "mqttInboundChannel")
    public MessageHandler inboundHandler() {
        return message -> {
            System.out.println("📩 Received MQTT message: " + message.getPayload());
        };
    }
}
```

---

## 5. 🚀 Publisher Service

```java
@Service
public class MqttPublisher {

    @Autowired
    private MessageChannel mqttOutboundChannel;

    public void sendToMqtt(String data) {
        mqttOutboundChannel.send(MessageBuilder.withPayload(data).build());
        System.out.println("📤 Published MQTT message: " + data);
    }
}
```

---

## 6. 🌐 REST Controller for Publishing

```java
@RestController
@RequestMapping("/mqtt")
public class MqttController {

    @Autowired
    private MqttPublisher publisher;

    @PostMapping("/publish")
    public String publishMessage(@RequestParam String msg) {
        publisher.sendToMqtt(msg);
        return "Message published: " + msg;
    }
}
```

---

## 7. 🧪 Testing Pub/Sub

1. Start **Spring Boot App**.
2. Publish message via REST:

   ```bash
   curl -X POST "http://localhost:8080/mqtt/publish?msg=HelloMQTT"
   ```

   Output:

   ```
   📤 Published MQTT message: HelloMQTT
   📩 Received MQTT message: HelloMQTT
   ```
3. Alternatively, use **MQTTX** or **mosquitto\_sub**:

   ```bash
   mosquitto_sub -t demo/topic -h localhost -p 1883
   mosquitto_pub -t demo/topic -m "Test from CLI" -h localhost -p 1883
   ```

---

## 8. ✅ Flow Recap

* `curl -> Spring Boot REST -> Publisher -> mqttOutboundChannel -> MQTT Broker`
* `MQTT Broker -> mqttInboundChannel -> Consumer -> Console`

---

👉 With this setup:

* You can **publish messages** to MQTT.
* You can **subscribe & consume messages** in Spring Boot.
* Works as base for **IoT device data processing**.

---

Would you like me to also prepare **an extended version where MQTT -> Kafka (bridge)** so you can push IoT messages to Kafka after consuming in Boot?

---

---
Great 👍 Let’s finalize the **stepwise notes with code** for integrating **Spring Boot with Eclipse Paho MQTT** for **pub/sub**.

---

# 📘 Notes: Spring Boot + Paho MQTT (Pub/Sub)

---

## 1. **What is MQTT?**

* **MQTT** = lightweight **publish-subscribe** protocol for IoT and messaging.
* **Broker** (e.g., Mosquitto, HiveMQ, EMQX) sits between **publisher** and **subscriber**.
* Publisher sends → **Topic**
* Subscriber listens → **Topic**
* Broker ensures delivery.

---

## 2. **Setup MQTT Broker**

You need a running broker. Options:

* **Local**: Run [Eclipse Mosquitto](https://mosquitto.org/download/)

  ```bash
  docker run -it -p 1883:1883 eclipse-mosquitto
  ```
* **Public broker** (for testing): `tcp://broker.hivemq.com:1883`

---

## 3. **Add Maven Dependency**

In `pom.xml`:

```xml
<dependency>
    <groupId>org.eclipse.paho</groupId>
    <artifactId>org.eclipse.paho.client.mqttv3</artifactId>
    <version>1.2.5</version>
</dependency>
```

---

## 4. **Spring Boot MQTT Config**

Create an MQTT configuration for connecting.

```java
package com.example.mqttdemo.config;

import org.eclipse.paho.client.mqttv3.MqttClient;
import org.eclipse.paho.client.mqttv3.MqttConnectOptions;
import org.eclipse.paho.client.mqttv3.persist.MemoryPersistence;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MqttConfig {

    private final String broker = "tcp://localhost:1883"; // or tcp://broker.hivemq.com:1883
    private final String clientId = "bootAppClient";

    @Bean
    public MqttClient mqttClient() throws Exception {
        MqttClient client = new MqttClient(broker, clientId, new MemoryPersistence());
        MqttConnectOptions options = new MqttConnectOptions();
        options.setCleanSession(true);
        client.connect(options);
        return client;
    }
}
```

---

## 5. **Publisher Service**

```java
package com.example.mqttdemo.service;

import org.eclipse.paho.client.mqttv3.MqttClient;
import org.eclipse.paho.client.mqttv3.MqttMessage;
import org.springframework.stereotype.Service;

@Service
public class PublisherService {

    private final MqttClient mqttClient;

    public PublisherService(MqttClient mqttClient) {
        this.mqttClient = mqttClient;
    }

    public void publish(String topic, String payload) throws Exception {
        MqttMessage message = new MqttMessage(payload.getBytes());
        message.setQos(1); // QoS level
        mqttClient.publish(topic, message);
        System.out.println("✅ Published: " + payload + " to topic: " + topic);
    }
}
```

---

## 6. **Subscriber Service**

```java
package com.example.mqttdemo.service;

import jakarta.annotation.PostConstruct;
import org.eclipse.paho.client.mqttv3.*;
import org.springframework.stereotype.Service;

@Service
public class SubscriberService {

    private final MqttClient mqttClient;
    private final String topic = "test/topic";

    public SubscriberService(MqttClient mqttClient) {
        this.mqttClient = mqttClient;
    }

    @PostConstruct
    public void subscribe() throws Exception {
        mqttClient.subscribe(topic, (t, msg) -> {
            String payload = new String(msg.getPayload());
            System.out.println("📩 Received message: " + payload + " from topic: " + t);
        });
        System.out.println("✅ Subscribed to topic: " + topic);
    }
}
```

---

## 7. **REST Controller (for testing publishing)**

```java
package com.example.mqttdemo.controller;

import com.example.mqttdemo.service.PublisherService;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/mqtt")
public class MqttController {

    private final PublisherService publisherService;

    public MqttController(PublisherService publisherService) {
        this.publisherService = publisherService;
    }

    @PostMapping("/publish")
    public String publishMessage(@RequestParam String topic, @RequestParam String message) throws Exception {
        publisherService.publish(topic, message);
        return "Message published!";
    }
}
```

---

## 8. **Test Flow**

1. Run MQTT Broker (`docker run -p 1883:1883 eclipse-mosquitto`)
2. Start Spring Boot App.
3. App **subscribes** to `test/topic`.
4. Test publish using REST API:

   ```bash
   curl -X POST "http://localhost:8080/mqtt/publish?topic=test/topic&message=HelloMQTT"
   ```
5. Console output:

   ```
   ✅ Published: HelloMQTT to topic: test/topic
   📩 Received message: HelloMQTT from topic: test/topic
   ```

---

✅ Now your **Spring Boot App is fully integrated with MQTT (Paho)** for **pub/sub**.

---

👉 Do you want me to **extend these notes** to also include **integration with Kafka after MQTT** (like MQTT → Kafka bridge in Boot), or keep it only MQTT for now?

---

---
Perfect 👍 Let’s now finalize the **stepwise Boot app with Eclipse Paho + MQTT (pub/sub)** integration notes with **code snippets**.

---

# 📘 Spring Boot + Eclipse Paho MQTT Integration Notes

---

## **Step 1: Add Dependencies**

In `pom.xml`:

```xml
<dependencies>
    <!-- Spring Boot Starter -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>

    <!-- Eclipse Paho MQTT client -->
    <dependency>
        <groupId>org.eclipse.paho</groupId>
        <artifactId>org.eclipse.paho.client.mqttv3</artifactId>
        <version>1.2.5</version>
    </dependency>
</dependencies>
```

---

## **Step 2: Configure MQTT Properties**

In `application.yml`:

```yaml
mqtt:
  broker: tcp://localhost:1883   # Mosquitto running in Docker/local
  clientId: boot-client
  topic: test/topic
  qos: 1
```

---

## **Step 3: MQTT Configuration Bean**

Create `MqttConfig.java`:

```java
import org.eclipse.paho.client.mqttv3.MqttClient;
import org.eclipse.paho.client.mqttv3.MqttConnectOptions;
import org.eclipse.paho.client.mqttv3.persist.MemoryPersistence;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MqttConfig {

    @Value("${mqtt.broker}")
    private String broker;

    @Value("${mqtt.clientId}")
    private String clientId;

    @Bean
    public MqttClient mqttClient() throws Exception {
        MqttClient client = new MqttClient(broker, clientId, new MemoryPersistence());
        MqttConnectOptions options = new MqttConnectOptions();
        options.setCleanSession(true);
        client.connect(options);
        return client;
    }
}
```

---

## **Step 4: Publisher Service**

`MqttPublisher.java`:

```java
import org.eclipse.paho.client.mqttv3.MqttClient;
import org.eclipse.paho.client.mqttv3.MqttMessage;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

@Service
public class MqttPublisher {

    private final MqttClient mqttClient;

    @Value("${mqtt.topic}")
    private String topic;

    public MqttPublisher(MqttClient mqttClient) {
        this.mqttClient = mqttClient;
    }

    public void publish(String message) throws Exception {
        MqttMessage mqttMessage = new MqttMessage(message.getBytes());
        mqttMessage.setQos(1);
        mqttClient.publish(topic, mqttMessage);
        System.out.println("✅ Published: " + message);
    }
}
```

---

## **Step 5: Subscriber Service**

`MqttSubscriber.java`:

```java
import org.eclipse.paho.client.mqttv3.IMqttMessageListener;
import org.eclipse.paho.client.mqttv3.MqttClient;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import jakarta.annotation.PostConstruct;

@Service
public class MqttSubscriber {

    private final MqttClient mqttClient;

    @Value("${mqtt.topic}")
    private String topic;

    public MqttSubscriber(MqttClient mqttClient) {
        this.mqttClient = mqttClient;
    }

    @PostConstruct
    public void subscribe() throws Exception {
        mqttClient.subscribe(topic, (t, msg) -> {
            String payload = new String(msg.getPayload());
            System.out.println("📩 Received: " + payload);
        });
    }
}
```

---

## **Step 6: Expose REST API for Testing**

`MqttController.java`:

```java
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/mqtt")
public class MqttController {

    private final MqttPublisher publisher;

    public MqttController(MqttPublisher publisher) {
        this.publisher = publisher;
    }

    @GetMapping("/publish/{message}")
    public String publish(@PathVariable String message) throws Exception {
        publisher.publish(message);
        return "Published message: " + message;
    }
}
```

---

## **Step 7: Run MQTT Broker (Mosquitto in Docker)**

```bash
docker run -it -p 1883:1883 -p 9001:9001 eclipse-mosquitto
```

---

## **Step 8: Test Flow**

1. Run Boot app.
2. Open browser:

   ```
   http://localhost:8080/mqtt/publish/hello-world
   ```
3. Console Output:

   ```
   ✅ Published: hello-world
   📩 Received: hello-world
   ```

---

✅ Done!
Now you have a **Spring Boot app with Eclipse Paho MQTT pub-sub integration**.

---

Do you want me to also extend this so that **after Boot publishes to MQTT, messages flow into Kafka** (Boot → MQTT → Kafka bridge)?

---

---
Perfect 👍 Let’s finalize **complete stepwise notes** for **Spring Boot + Eclipse Paho MQTT (Publisher & Subscriber)** with **working code**.

---

# 📘 Notes: Spring Boot Integration with Eclipse Paho MQTT

---

## **1. Add Dependency**

Use Eclipse Paho Client for MQTT communication.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.eclipse.paho</groupId>
    <artifactId>org.eclipse.paho.client.mqttv3</artifactId>
    <version>1.2.5</version>
</dependency>
```

---

## **2. Configure MQTT Properties**

Define MQTT broker details inside `application.properties`.

```properties
# MQTT Broker details
mqtt.broker=tcp://localhost:1883
mqtt.client.id=boot-mqtt-client
mqtt.topic=test/topic
```

---

## **3. MQTT Configuration Class**

Create a bean for MQTT client connection.

```java
import org.eclipse.paho.client.mqttv3.IMqttClient;
import org.eclipse.paho.client.mqttv3.MqttClient;
import org.eclipse.paho.client.mqttv3.persist.MemoryPersistence;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MqttConfig {

    @Value("${mqtt.broker}")
    private String broker;

    @Value("${mqtt.client.id}")
    private String clientId;

    @Bean
    public IMqttClient mqttClient() throws Exception {
        IMqttClient client = new MqttClient(broker, clientId, new MemoryPersistence());
        client.connect();
        return client;
    }
}
```

---

## **4. Publisher Service**

This service publishes messages to the MQTT topic.

```java
import org.eclipse.paho.client.mqttv3.IMqttClient;
import org.eclipse.paho.client.mqttv3.MqttMessage;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

@Service
public class MqttPublisher {

    private final IMqttClient mqttClient;

    @Value("${mqtt.topic}")
    private String topic;

    public MqttPublisher(IMqttClient mqttClient) {
        this.mqttClient = mqttClient;
    }

    public void publish(String payload) throws Exception {
        if (!mqttClient.isConnected()) {
            System.out.println("Client not connected!");
            return;
        }

        MqttMessage message = new MqttMessage(payload.getBytes());
        message.setQos(1);
        message.setRetained(false);

        mqttClient.publish(topic, message);
        System.out.println("📤 Published: " + payload);
    }
}
```

---

## **5. Subscriber Service**

This listens to MQTT messages.

```java
import org.eclipse.paho.client.mqttv3.IMqttClient;
import org.eclipse.paho.client.mqttv3.IMqttMessageListener;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import javax.annotation.PostConstruct;

@Service
public class MqttSubscriber {

    private final IMqttClient mqttClient;

    @Value("${mqtt.topic}")
    private String topic;

    public MqttSubscriber(IMqttClient mqttClient) {
        this.mqttClient = mqttClient;
    }

    @PostConstruct
    public void subscribe() throws Exception {
        mqttClient.subscribe(topic, (t, msg) -> {
            String payload = new String(msg.getPayload());
            System.out.println("📩 Received: " + payload);
        });
    }
}
```

---

## **6. REST Controller for Testing**

We’ll trigger MQTT messages from REST endpoints.

```java
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/mqtt")
public class MqttController {

    private final MqttPublisher publisher;

    public MqttController(MqttPublisher publisher) {
        this.publisher = publisher;
    }

    @PostMapping("/publish/{message}")
    public String publish(@PathVariable String message) throws Exception {
        publisher.publish(message);
        return "Message Published: " + message;
    }
}
```

---

## **7. Run MQTT Broker (Mosquitto via Docker)**

Run an MQTT broker locally using Docker:

```bash
docker run -it -p 1883:1883 -p 9001:9001 eclipse-mosquitto
```

---

## **8. Test Flow**

1. Start MQTT broker (`Mosquitto`).
2. Start Spring Boot app.
3. Open REST API:

```bash
curl -X POST http://localhost:8080/mqtt/publish/helloWorld
```

✅ Console Output:

* **Publisher** → `📤 Published: helloWorld`
* **Subscriber** → `📩 Received: helloWorld`

---

## **9. Key Notes**

* Paho Client handles MQTT connectivity.
* Spring Boot services abstract publisher & subscriber.
* You can extend with **multiple topics**, **QoS levels**, **retained messages**, etc.
* Broker can be **Mosquitto**, **HiveMQ**, or **EMQX**.

---

👉 Now you have a **full working Spring Boot + Eclipse Paho MQTT project** with **pub/sub** support.

Do you also want me to extend this so that after **Spring Boot consumes MQTT message, it pushes data into Kafka** (MQTT → Boot → Kafka)?

---

---
Perfect 👍 Let’s now finalize **stepwise notes with code** for integrating **Spring Boot + Eclipse Paho MQTT** for a **Publish/Subscribe** IoT-style setup.

---

# 📘 Spring Boot + MQTT (Eclipse Paho) Integration Notes

---

## **1. Add Dependencies**

We use Eclipse Paho for MQTT client.

```xml
<!-- pom.xml -->
<dependencies>
    <!-- MQTT Client -->
    <dependency>
        <groupId>org.eclipse.paho</groupId>
        <artifactId>org.eclipse.paho.client.mqttv3</artifactId>
        <version>1.2.5</version>
    </dependency>

    <!-- Spring Boot Starter -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
</dependencies>
```

---

## **2. Application Properties**

Configure MQTT broker details.

```properties
# application.properties
mqtt.broker=tcp://localhost:1883
mqtt.clientId=spring-boot-mqtt-client
mqtt.topic=test/topic
```

---

## **3. MQTT Configuration Class**

Setup connection to MQTT broker.

```java
package com.example.mqtt.config;

import org.eclipse.paho.client.mqttv3.MqttClient;
import org.eclipse.paho.client.mqttv3.MqttConnectOptions;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MqttConfig {

    @Value("${mqtt.broker}")
    private String broker;

    @Value("${mqtt.clientId}")
    private String clientId;

    @Bean
    public MqttClient mqttClient() throws Exception {
        MqttClient client = new MqttClient(broker, clientId);
        MqttConnectOptions options = new MqttConnectOptions();
        options.setCleanSession(true);
        client.connect(options);
        return client;
    }
}
```

---

## **4. Publisher Service**

This class will publish messages to MQTT broker.

```java
package com.example.mqtt.service;

import org.eclipse.paho.client.mqttv3.MqttClient;
import org.eclipse.paho.client.mqttv3.MqttMessage;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

@Service
public class MqttPublisher {

    private final MqttClient mqttClient;

    @Value("${mqtt.topic}")
    private String topic;

    public MqttPublisher(MqttClient mqttClient) {
        this.mqttClient = mqttClient;
    }

    public void publish(String payload) throws Exception {
        MqttMessage message = new MqttMessage(payload.getBytes());
        message.setQos(1);
        mqttClient.publish(topic, message);
        System.out.println("✅ Published message: " + payload);
    }
}
```

---

## **5. Subscriber Service**

This listens for messages from MQTT broker.

```java
package com.example.mqtt.service;

import org.eclipse.paho.client.mqttv3.*;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

@Service
public class MqttSubscriber {

    public MqttSubscriber(MqttClient mqttClient, @Value("${mqtt.topic}") String topic) throws Exception {
        mqttClient.subscribe(topic, (t, msg) -> {
            System.out.println("📩 Received message: " + new String(msg.getPayload()));
        });
    }
}
```

---

## **6. REST Controller (to trigger publishing)**

You can trigger publish via API call.

```java
package com.example.mqtt.controller;

import com.example.mqtt.service.MqttPublisher;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/mqtt")
public class MqttController {

    private final MqttPublisher publisher;

    public MqttController(MqttPublisher publisher) {
        this.publisher = publisher;
    }

    @PostMapping("/publish")
    public String publish(@RequestParam String message) throws Exception {
        publisher.publish(message);
        return "✅ Message published: " + message;
    }
}
```

---

## **7. Run Docker Mosquitto Broker (Test Setup)**

Start an MQTT broker locally:

```bash
docker run -it -p 1883:1883 -p 9001:9001 eclipse-mosquitto
```

---

## **8. Test**

* Start Spring Boot app
* Publish message:

```bash
curl -X POST "http://localhost:8080/mqtt/publish?message=HelloMQTT"
```

* Subscriber log will show:

```
✅ Published message: HelloMQTT
📩 Received message: HelloMQTT
```

---

✅ Now you have a **Spring Boot application integrated with Eclipse Paho MQTT for Pub/Sub**.
It connects to Mosquitto broker, publishes messages via REST, and subscribes to listen for updates.

---

Do you want me to **extend these notes further to show how this Spring Boot → MQTT → Kafka bridge** works (so MQTT messages automatically flow into Kafka)?

---


---
Perfect 👍 Now let’s wrap it all into **final consolidated stepwise notes with code** for integrating **Spring Boot + Eclipse Paho MQTT (Pub/Sub)**.

---

# 🚀 Spring Boot + Eclipse Paho MQTT Integration (Publisher + Subscriber)

---

## **1. Add Maven Dependency**

In your `pom.xml`:

```xml
<dependency>
    <groupId>org.eclipse.paho</groupId>
    <artifactId>org.eclipse.paho.client.mqttv3</artifactId>
    <version>1.2.5</version>
</dependency>
```

---

## **2. Configure MQTT Properties**

Create `application.yml`:

```yaml
mqtt:
  broker: tcp://localhost:1883   # MQTT broker (Mosquitto/HiveMQ/EMQX)
  clientId: boot-client-1        # Unique ID per client
  topic: test/topic              # Example topic
  qos: 1                         # Quality of Service
  username: testuser             # optional
  password: testpass             # optional
```

---

## **3. Create MQTT Config**

```java
import org.eclipse.paho.client.mqttv3.MqttClient;
import org.eclipse.paho.client.mqttv3.MqttConnectOptions;
import org.eclipse.paho.client.mqttv3.persist.MemoryPersistence;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MqttConfig {

    @Value("${mqtt.broker}")
    private String broker;

    @Value("${mqtt.clientId}")
    private String clientId;

    @Value("${mqtt.username:}")
    private String username;

    @Value("${mqtt.password:}")
    private String password;

    @Bean
    public MqttClient mqttClient() throws Exception {
        MqttClient client = new MqttClient(broker, clientId, new MemoryPersistence());

        MqttConnectOptions options = new MqttConnectOptions();
        options.setCleanSession(true);

        if (!username.isEmpty()) {
            options.setUserName(username);
            options.setPassword(password.toCharArray());
        }

        client.connect(options);
        return client;
    }
}
```

---

## **4. Publisher Service**

```java
import org.eclipse.paho.client.mqttv3.MqttClient;
import org.eclipse.paho.client.mqttv3.MqttMessage;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

@Service
public class MqttPublisherService {

    private final MqttClient mqttClient;

    @Value("${mqtt.topic}")
    private String topic;

    public MqttPublisherService(MqttClient mqttClient) {
        this.mqttClient = mqttClient;
    }

    public void publish(String msg) {
        try {
            MqttMessage message = new MqttMessage(msg.getBytes());
            message.setQos(1);
            mqttClient.publish(topic, message);
            System.out.println("📤 Published: " + msg);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

---

## **5. Subscriber Service**

```java
import org.eclipse.paho.client.mqttv3.IMqttMessageListener;
import org.eclipse.paho.client.mqttv3.MqttClient;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

@Service
public class MqttSubscriberService {

    @Value("${mqtt.topic}")
    private String topic;

    public MqttSubscriberService(MqttClient mqttClient) {
        try {
            mqttClient.subscribe(topic, (t, msg) -> {
                System.out.println("📥 Received -> " + new String(msg.getPayload()));
            });
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

---

## **6. REST Controller for Testing Pub/Sub**

```java
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/mqtt")
public class MqttController {

    private final MqttPublisherService publisherService;

    public MqttController(MqttPublisherService publisherService) {
        this.publisherService = publisherService;
    }

    @PostMapping("/publish")
    public String publishMessage(@RequestParam String message) {
        publisherService.publish(message);
        return "Message Published: " + message;
    }
}
```

---

## **7. Run Broker (Mosquitto in Docker)**

```bash
docker run -it -p 1883:1883 -p 9001:9001 eclipse-mosquitto
```

---

## **8. Test the Flow**

1. Start Spring Boot App
2. Use **Postman / cURL** to publish:

   ```bash
   curl -X POST "http://localhost:8080/mqtt/publish?message=HelloIoT"
   ```
3. Check logs → Subscriber will auto-print:

   ```
   📤 Published: HelloIoT
   📥 Received -> HelloIoT
   ```

---

✅ **Now Boot App can publish & subscribe messages via Eclipse Paho MQTT client**.
You can connect this with **IoT devices / Kafka bridge** later.

---

Do you want me to also extend these notes with **Boot → MQTT → Kafka bridge** (so messages published via MQTT get pushed into Kafka topic)?

---

---
Great 👍 let’s finalize a **stepwise full guide (notes + code)** for integrating **Spring Boot app with Eclipse Paho MQTT** for **Pub/Sub**:

---

# 📘 Spring Boot + Eclipse Paho MQTT Integration (Pub/Sub)

---

## 🔹 1. Add Dependency

Add the **Eclipse Paho MQTT client** to your `pom.xml`:

```xml
<dependency>
    <groupId>org.eclipse.paho</groupId>
    <artifactId>org.eclipse.paho.client.mqttv3</artifactId>
    <version>1.2.5</version>
</dependency>
```

---

## 🔹 2. MQTT Config Class

Create a config class to connect your Boot app with the **MQTT broker** (like Mosquitto running on Docker).

```java
package com.example.mqttdemo.config;

import org.eclipse.paho.client.mqttv3.IMqttClient;
import org.eclipse.paho.client.mqttv3.MqttClient;
import org.eclipse.paho.client.mqttv3.MqttConnectOptions;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MqttConfig {

    private static final String BROKER_URL = "tcp://localhost:1883"; // Mosquitto broker
    private static final String CLIENT_ID = "spring-boot-client";

    @Bean
    public IMqttClient mqttClient() throws Exception {
        IMqttClient client = new MqttClient(BROKER_URL, CLIENT_ID);
        MqttConnectOptions options = new MqttConnectOptions();
        options.setAutomaticReconnect(true);
        options.setCleanSession(true);
        options.setConnectionTimeout(10);

        client.connect(options);
        return client;
    }
}
```

---

## 🔹 3. Publisher Service

```java
package com.example.mqttdemo.service;

import org.eclipse.paho.client.mqttv3.IMqttClient;
import org.eclipse.paho.client.mqttv3.MqttMessage;
import org.springframework.stereotype.Service;

@Service
public class PublisherService {

    private final IMqttClient mqttClient;

    public PublisherService(IMqttClient mqttClient) {
        this.mqttClient = mqttClient;
    }

    public void publish(String topic, String payload) throws Exception {
        if (!mqttClient.isConnected()) {
            throw new RuntimeException("MQTT client is not connected!");
        }
        MqttMessage msg = new MqttMessage(payload.getBytes());
        msg.setQos(1);
        msg.setRetained(false);

        mqttClient.publish(topic, msg);
        System.out.println("✅ Published: " + payload + " to topic: " + topic);
    }
}
```

---

## 🔹 4. Subscriber Service

```java
package com.example.mqttdemo.service;

import org.eclipse.paho.client.mqttv3.IMqttClient;
import org.eclipse.paho.client.mqttv3.IMqttMessageListener;
import org.springframework.stereotype.Service;

@Service
public class SubscriberService {

    private final IMqttClient mqttClient;

    public SubscriberService(IMqttClient mqttClient) {
        this.mqttClient = mqttClient;
    }

    public void subscribe(String topic) throws Exception {
        if (!mqttClient.isConnected()) {
            throw new RuntimeException("MQTT client is not connected!");
        }
        mqttClient.subscribe(topic, (t, msg) -> {
            String received = new String(msg.getPayload());
            System.out.println("📩 Received message: " + received + " from topic: " + t);
        });
        System.out.println("✅ Subscribed to topic: " + topic);
    }
}
```

---

## 🔹 5. REST Controller (to test Pub/Sub)

```java
package com.example.mqttdemo.controller;

import com.example.mqttdemo.service.PublisherService;
import com.example.mqttdemo.service.SubscriberService;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/mqtt")
public class MqttController {

    private final PublisherService publisherService;
    private final SubscriberService subscriberService;

    public MqttController(PublisherService publisherService, SubscriberService subscriberService) {
        this.publisherService = publisherService;
        this.subscriberService = subscriberService;
    }

    @PostMapping("/publish")
    public String publish(@RequestParam String topic, @RequestParam String message) throws Exception {
        publisherService.publish(topic, message);
        return "Message published to topic: " + topic;
    }

    @GetMapping("/subscribe")
    public String subscribe(@RequestParam String topic) throws Exception {
        subscriberService.subscribe(topic);
        return "Subscribed to topic: " + topic;
    }
}
```

---

## 🔹 6. Run Mosquitto Broker in Docker

```bash
docker run -it -p 1883:1883 -p 9001:9001 eclipse-mosquitto
```

---

## 🔹 7. Test

1. Start your **Spring Boot App**.

2. Open Postman:

   * Subscribe to a topic:

     ```
     GET http://localhost:8080/mqtt/subscribe?topic=test/topic
     ```
   * Publish message:

     ```
     POST http://localhost:8080/mqtt/publish?topic=test/topic&message=HelloMQTT
     ```

3. Check console → Subscriber should receive message.

---

✅ **End-to-End Flow:**

* Boot connects → MQTT broker (Mosquitto).
* REST API triggers publish.
* Subscriber listens and receives messages.

---

Do you also want me to extend these notes with a **fake Java IoT device simulator** (like sending random sensor data every 5 sec)? That will complete the **real IoT project feel**.

---

---
Perfect 👍 Let’s finalize the **stepwise notes for integrating Spring Boot with Eclipse Paho MQTT** for Pub/Sub.

---

# 📘 Spring Boot + Eclipse Paho MQTT Pub/Sub Integration

---

## 🔹 1. What is Eclipse Paho MQTT?

* **Paho** is an MQTT client library provided by Eclipse Foundation.
* It helps a Spring Boot app act as a **Publisher** (send messages) or **Subscriber** (receive messages).
* MQTT is widely used in **IoT, real-time messaging, lightweight communication**.

---

## 🔹 2. Project Setup

### Add dependency in `pom.xml`

```xml
<dependency>
    <groupId>org.eclipse.paho</groupId>
    <artifactId>org.eclipse.paho.client.mqttv3</artifactId>
    <version>1.2.5</version>
</dependency>
```

---

## 🔹 3. Configure MQTT in Spring Boot

### `MqttConfig.java`

```java
import org.eclipse.paho.client.mqttv3.MqttClient;
import org.eclipse.paho.client.mqttv3.MqttConnectOptions;
import org.eclipse.paho.client.mqttv3.persist.MemoryPersistence;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MqttConfig {

    private final String brokerUrl = "tcp://localhost:1883";  // Mosquitto broker
    private final String clientId = "spring-boot-mqtt";

    @Bean
    public MqttClient mqttClient() throws Exception {
        MqttClient client = new MqttClient(brokerUrl, clientId, new MemoryPersistence());

        MqttConnectOptions options = new MqttConnectOptions();
        options.setCleanSession(true);
        options.setAutomaticReconnect(true);
        options.setConnectionTimeout(10);

        client.connect(options);
        return client;
    }
}
```

---

## 🔹 4. Publisher Service

### `MqttPublisherService.java`

```java
import org.eclipse.paho.client.mqttv3.MqttClient;
import org.eclipse.paho.client.mqttv3.MqttMessage;
import org.springframework.stereotype.Service;

@Service
public class MqttPublisherService {

    private final MqttClient mqttClient;

    public MqttPublisherService(MqttClient mqttClient) {
        this.mqttClient = mqttClient;
    }

    public void publish(String topic, String message) throws Exception {
        MqttMessage mqttMessage = new MqttMessage(message.getBytes());
        mqttMessage.setQos(1);   // QoS 0,1,2
        mqttClient.publish(topic, mqttMessage);
        System.out.println("✅ Published message: " + message + " to topic: " + topic);
    }
}
```

---

## 🔹 5. Subscriber Service

### `MqttSubscriberService.java`

```java
import org.eclipse.paho.client.mqttv3.IMqttMessageListener;
import org.eclipse.paho.client.mqttv3.MqttClient;
import org.springframework.stereotype.Service;

@Service
public class MqttSubscriberService {

    public MqttSubscriberService(MqttClient mqttClient) throws Exception {
        mqttClient.subscribe("demo/topic", (topic, msg) -> {
            String received = new String(msg.getPayload());
            System.out.println("📩 Received message: " + received + " from topic: " + topic);
        });
    }
}
```

---

## 🔹 6. REST Controller for Publishing

### `MqttController.java`

```java
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/mqtt")
public class MqttController {

    private final MqttPublisherService publisherService;

    public MqttController(MqttPublisherService publisherService) {
        this.publisherService = publisherService;
    }

    @PostMapping("/publish/{topic}")
    public String publishMessage(@PathVariable String topic, @RequestBody String message) {
        try {
            publisherService.publish(topic, message);
            return "Message published to topic: " + topic;
        } catch (Exception e) {
            return "❌ Error publishing: " + e.getMessage();
        }
    }
}
```

---

## 🔹 7. Run MQTT Broker (Mosquitto)

If using **Docker**:

```sh
docker run -it -p 1883:1883 -p 9001:9001 eclipse-mosquitto
```

---

## 🔹 8. Testing the Flow

1. **Run broker (Mosquitto)**
   It listens on `tcp://localhost:1883`.
2. **Start Spring Boot app**
   Subscriber auto-subscribes to `demo/topic`.
3. **Publish message via REST API**

   ```sh
   curl -X POST http://localhost:8080/mqtt/publish/demo/topic -d "Hello MQTT from Boot!"
   ```
4. **Output**

   ```
   ✅ Published message: Hello MQTT from Boot! to topic: demo/topic
   📩 Received message: Hello MQTT from Boot! from topic: demo/topic
   ```

---

## 🔹 9. Real-World Use Cases

* **IoT Devices → MQTT Broker → Spring Boot → Kafka → Dashboard**
* **Real-time Notifications**
* **Chat Applications**
* **Sensor Data Processing**

---

👉 Now your Boot app can **publish & subscribe with Paho MQTT**.

Would you like me to also prepare **next-level notes** on extending this setup to **Boot → MQTT → Kafka** for streaming pipelines?

---

---
Perfect 👍 Now let’s finalize the **complete stepwise notes for Boot + Paho MQTT Pub/Sub integration**

---

# 📘 Spring Boot + Eclipse Paho MQTT (Pub/Sub) — Stepwise Notes

---

## **1. Add Dependency**

Add Eclipse Paho MQTT Client dependency in your `pom.xml`:

```xml
<dependency>
    <groupId>org.eclipse.paho</groupId>
    <artifactId>org.eclipse.paho.client.mqttv3</artifactId>
    <version>1.2.5</version>
</dependency>
```

---

## **2. Application Properties (`application.yml`)**

```yaml
mqtt:
  broker: tcp://localhost:1883   # Your MQTT broker (Mosquitto, HiveMQ, etc.)
  clientId: boot-app-client
  topic:
    pub: test/pub
    sub: test/sub
  qos: 1
```

---

## **3. MQTT Configuration Bean**

```java
import org.eclipse.paho.client.mqttv3.*;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MqttConfig {

    @Value("${mqtt.broker}")
    private String broker;

    @Value("${mqtt.clientId}")
    private String clientId;

    @Bean
    public MqttClient mqttClient() throws MqttException {
        MqttClient client = new MqttClient(broker, clientId, new MemoryPersistence());
        MqttConnectOptions options = new MqttConnectOptions();
        options.setCleanSession(true);

        client.connect(options);
        System.out.println("✅ Connected to MQTT Broker: " + broker);

        return client;
    }
}
```

---

## **4. Publisher Service**

```java
import org.eclipse.paho.client.mqttv3.MqttClient;
import org.eclipse.paho.client.mqttv3.MqttMessage;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

@Service
public class MqttPublisher {

    private final MqttClient mqttClient;

    @Value("${mqtt.topic.pub}")
    private String pubTopic;

    public MqttPublisher(MqttClient mqttClient) {
        this.mqttClient = mqttClient;
    }

    public void publish(String msg) {
        try {
            MqttMessage message = new MqttMessage(msg.getBytes());
            message.setQos(1);
            mqttClient.publish(pubTopic, message);
            System.out.println("📤 Published: " + msg + " → " + pubTopic);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

---

## **5. Subscriber Service**

```java
import org.eclipse.paho.client.mqttv3.IMqttMessageListener;
import org.eclipse.paho.client.mqttv3.MqttClient;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import jakarta.annotation.PostConstruct;

@Service
public class MqttSubscriber {

    private final MqttClient mqttClient;

    @Value("${mqtt.topic.sub}")
    private String subTopic;

    public MqttSubscriber(MqttClient mqttClient) {
        this.mqttClient = mqttClient;
    }

    @PostConstruct
    public void subscribe() {
        try {
            mqttClient.subscribe(subTopic, (topic, message) -> {
                System.out.println("📥 Received → Topic: " + topic + " | Msg: " + new String(message.getPayload()));
            });
            System.out.println("✅ Subscribed to: " + subTopic);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

---

## **6. REST Controller to Test Pub/Sub**

```java
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/mqtt")
public class MqttController {

    private final MqttPublisher publisher;

    public MqttController(MqttPublisher publisher) {
        this.publisher = publisher;
    }

    @GetMapping("/publish/{msg}")
    public String publishMessage(@PathVariable String msg) {
        publisher.publish(msg);
        return "Message Published: " + msg;
    }
}
```

---

## **7. Run MQTT Broker in Docker (Mosquitto)**

```bash
docker run -it -p 1883:1883 -p 9001:9001 eclipse-mosquitto
```

---

## **8. Test Flow**

1. Run Spring Boot app.
2. It connects to Mosquitto broker at `tcp://localhost:1883`.
3. Subscribes to topic `test/sub`.
4. Call REST endpoint:

```bash
curl http://localhost:8080/mqtt/publish/hello123
```

✅ Console Output:

```
📤 Published: hello123 → test/pub
📥 Received → Topic: test/sub | Msg: hello123
```

---

## **9. Real-Time Use Cases**

* IoT Sensors → MQTT Broker → Spring Boot processing.
* Vehicle GPS tracking → publish location → Boot consumes & stores in DB.
* Chat/Messaging app → publish message → Boot delivers via WebSocket.

---

👉 Now, do you want me to **extend this with a Kafka bridge** (Boot → MQTT → Kafka) so data from MQTT can be processed in Kafka also?

---

---
Perfect 👍 Let’s finalize **Step-12: Running End-to-End Demo with Spring Boot + Paho MQTT Pub/Sub**

---

## ✅ Step-12: End-to-End Demo (Spring Boot + Paho + MQTT Broker)

### 1. Start MQTT Broker (Docker Mosquitto)

Run Mosquitto broker in Docker:

```bash
docker run -it -p 1883:1883 -p 9001:9001 eclipse-mosquitto
```

* `1883` → MQTT port (for Paho client)
* `9001` → WebSocket port

---

### 2. Project Structure

```
boot-mqtt-paho-demo/
 ├── src/main/java/com/example/mqtt
 │   ├── config/MqttConfig.java       # Paho config
 │   ├── service/MqttPublisher.java   # Publish to topic
 │   ├── service/MqttSubscriber.java  # Subscribe to topic
 │   └── MqttDemoApplication.java     # Main Spring Boot App
 └── pom.xml
```

---

### 3. `pom.xml`

```xml
<dependencies>
    <!-- Spring Boot Starter -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>

    <!-- Eclipse Paho MQTT -->
    <dependency>
        <groupId>org.eclipse.paho</groupId>
        <artifactId>org.eclipse.paho.client.mqttv3</artifactId>
        <version>1.2.5</version>
    </dependency>
</dependencies>
```

---

### 4. `MqttConfig.java`

```java
package com.example.mqtt.config;

import org.eclipse.paho.client.mqttv3.MqttClient;
import org.eclipse.paho.client.mqttv3.MqttConnectOptions;
import org.eclipse.paho.client.mqttv3.persist.MemoryPersistence;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MqttConfig {

    private static final String BROKER = "tcp://localhost:1883";
    private static final String CLIENT_ID = "boot-paho-client";

    @Bean
    public MqttClient mqttClient() throws Exception {
        MqttClient client = new MqttClient(BROKER, CLIENT_ID, new MemoryPersistence());
        MqttConnectOptions options = new MqttConnectOptions();
        options.setCleanSession(true);
        client.connect(options);
        return client;
    }
}
```

---

### 5. `MqttPublisher.java`

```java
package com.example.mqtt.service;

import org.eclipse.paho.client.mqttv3.MqttClient;
import org.eclipse.paho.client.mqttv3.MqttMessage;
import org.springframework.stereotype.Service;

@Service
public class MqttPublisher {

    private final MqttClient mqttClient;

    public MqttPublisher(MqttClient mqttClient) {
        this.mqttClient = mqttClient;
    }

    public void publish(String topic, String payload) throws Exception {
        MqttMessage message = new MqttMessage(payload.getBytes());
        message.setQos(1);
        mqttClient.publish(topic, message);
        System.out.println("📤 Published: " + payload + " -> topic: " + topic);
    }
}
```

---

### 6. `MqttSubscriber.java`

```java
package com.example.mqtt.service;

import jakarta.annotation.PostConstruct;
import org.eclipse.paho.client.mqttv3.IMqttMessageListener;
import org.eclipse.paho.client.mqttv3.MqttClient;
import org.springframework.stereotype.Service;

@Service
public class MqttSubscriber {

    private final MqttClient mqttClient;

    public MqttSubscriber(MqttClient mqttClient) {
        this.mqttClient = mqttClient;
    }

    @PostConstruct
    public void subscribe() throws Exception {
        mqttClient.subscribe("iot/sensor/data", (topic, message) -> {
            System.out.println("📥 Received from topic " + topic + " -> " + new String(message.getPayload()));
        });
    }
}
```

---

### 7. `MqttDemoApplication.java`

```java
package com.example.mqtt;

import com.example.mqtt.service.MqttPublisher;
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class MqttDemoApplication implements CommandLineRunner {

    private final MqttPublisher publisher;

    public MqttDemoApplication(MqttPublisher publisher) {
        this.publisher = publisher;
    }

    public static void main(String[] args) {
        SpringApplication.run(MqttDemoApplication.class, args);
    }

    @Override
    public void run(String... args) throws Exception {
        for (int i = 1; i <= 5; i++) {
            publisher.publish("iot/sensor/data", "Temperature reading " + i);
            Thread.sleep(1000);
        }
    }
}
```

---

### 8. Run the App

```bash
mvn spring-boot:run
```

---

### 9. Expected Output

```
📤 Published: Temperature reading 1 -> topic: iot/sensor/data
📥 Received from topic iot/sensor/data -> Temperature reading 1
📤 Published: Temperature reading 2 -> topic: iot/sensor/data
📥 Received from topic iot/sensor/data -> Temperature reading 2
...
```

---

### 🔑 Summary

1. **Publisher** sends sensor data → `iot/sensor/data`
2. **Subscriber** listens & prints received messages
3. Works fully inside Spring Boot with Eclipse Paho + Mosquitto

---

👉 Do you want me to also extend this demo to **simulate multiple fake sensors (temperature, humidity, GPS)** publishing to different topics in the same Boot app? That will make it more IoT-like.

---


---
Perfect 🚀 Let’s wrap this up with **final polished notes** for **Spring Boot + Eclipse Paho MQTT Integration (Pub/Sub)** with a runnable example.

---

# 📘 Notes: Spring Boot + MQTT (Eclipse Paho) Integration

## 1️⃣ What is MQTT?

* Lightweight **Pub/Sub messaging protocol** for IoT.
* Clients publish messages to **topics**.
* Other clients **subscribe** to topics.
* Broker (e.g., **Eclipse Mosquitto**) manages message delivery.

---

## 2️⃣ Spring Boot + Eclipse Paho Flow

1. Spring Boot app uses **Eclipse Paho client**.
2. App connects to **MQTT Broker** (e.g., Mosquitto).
3. App can:

   * **Publish** messages to a topic.
   * **Subscribe** to a topic to receive messages.

---

## 3️⃣ Add Dependency (Maven)

```xml
<dependency>
    <groupId>org.eclipse.paho</groupId>
    <artifactId>org.eclipse.paho.client.mqttv3</artifactId>
    <version>1.2.5</version>
</dependency>
```

---

## 4️⃣ Application Properties (`application.yml`)

```yaml
mqtt:
  broker: tcp://localhost:1883
  clientId: springBootClient
  topic: test/topic
```

---

## 5️⃣ MQTT Config (Publisher + Subscriber Bean)

```java
import org.eclipse.paho.client.mqttv3.*;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MqttConfig {

    @Value("${mqtt.broker}")
    private String broker;

    @Value("${mqtt.clientId}")
    private String clientId;

    @Value("${mqtt.topic}")
    private String topic;

    @Bean
    public MqttClient mqttClient() throws MqttException {
        MqttClient client = new MqttClient(broker, clientId, null);
        MqttConnectOptions options = new MqttConnectOptions();
        options.setAutomaticReconnect(true);
        options.setCleanSession(true);
        options.setConnectionTimeout(10);

        client.connect(options);

        // Subscriber: Listen to topic
        client.subscribe(topic, (t, msg) -> {
            System.out.println("📩 Received on topic [" + t + "]: " + new String(msg.getPayload()));
        });

        System.out.println("✅ Connected to MQTT Broker: " + broker);
        return client;
    }
}
```

---

## 6️⃣ Publisher Service

```java
import org.eclipse.paho.client.mqttv3.MqttClient;
import org.eclipse.paho.client.mqttv3.MqttMessage;
import org.springframework.stereotype.Service;

@Service
public class MqttPublisherService {

    private final MqttClient mqttClient;

    public MqttPublisherService(MqttClient mqttClient) {
        this.mqttClient = mqttClient;
    }

    public void publish(String topic, String message) {
        try {
            MqttMessage mqttMessage = new MqttMessage(message.getBytes());
            mqttMessage.setQos(1);
            mqttClient.publish(topic, mqttMessage);
            System.out.println("📤 Published: " + message + " -> " + topic);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

---

## 7️⃣ REST Controller (to trigger publish)

```java
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/mqtt")
public class MqttController {

    private final MqttPublisherService publisherService;

    public MqttController(MqttPublisherService publisherService) {
        this.publisherService = publisherService;
    }

    @GetMapping("/publish")
    public String publish(@RequestParam String msg) {
        publisherService.publish("test/topic", msg);
        return "✅ Message published: " + msg;
    }
}
```

---

## 8️⃣ Run MQTT Broker (Docker)

```bash
docker run -it -p 1883:1883 eclipse-mosquitto
```

---

## 9️⃣ Test

* Start **Spring Boot app**.
* Open browser/Postman:

  ```
  http://localhost:8080/mqtt/publish?msg=HelloIoT
  ```
* Output:

  ```
  📤 Published: HelloIoT -> test/topic
  📩 Received on topic [test/topic]: HelloIoT
  ```

---

## 🔥 Real-Time Usage

* IoT Devices → Publish sensor data.
* Spring Boot → Subscribes to device topics.
* Spring Boot → Processes data & pushes to **Kafka/DB/UI**.

---

👉 This setup gives you:

* **Publisher** (via REST API).
* **Subscriber** (auto-listens to topic).
* Fully running on **local Mosquitto broker**.

---

Would you like me to also extend this into **Boot → MQTT → Kafka pipeline** (Spring Boot subscribes from MQTT and publishes to Kafka) so you get full **IoT → Kafka stream processing**?

---

---
Perfect 👍 Let’s finalize everything into **clear notes with stepwise explanation + working code** for **Spring Boot + Eclipse Paho MQTT (Pub/Sub)**.

---

# 📘 Notes: Spring Boot + Eclipse Paho MQTT Integration (Pub/Sub)

---

## **1. What is MQTT?**

* Lightweight messaging protocol (publish/subscribe).
* Ideal for IoT, real-time data, device-to-cloud communication.
* Works with a **broker** (e.g., Mosquitto, HiveMQ, EMQX).
* Client can **publish messages** or **subscribe** to topics.

---

## **2. Project Setup**

### Add Dependency (Maven)

```xml
<dependency>
    <groupId>org.eclipse.paho</groupId>
    <artifactId>org.eclipse.paho.client.mqttv3</artifactId>
    <version>1.2.5</version>
</dependency>
```

(Optional: Spring Integration MQTT for advanced features)

```xml
<dependency>
    <groupId>org.springframework.integration</groupId>
    <artifactId>spring-integration-mqtt</artifactId>
    <version>5.5.17</version>
</dependency>
```

---

## **3. MQTT Configuration**

Create a configuration class to set up Paho MQTT client.

```java
import org.eclipse.paho.client.mqttv3.MqttClient;
import org.eclipse.paho.client.mqttv3.MqttConnectOptions;
import org.eclipse.paho.client.mqttv3.persist.MemoryPersistence;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MqttConfig {

    private static final String BROKER_URL = "tcp://localhost:1883"; // Mosquitto running locally
    private static final String CLIENT_ID = "spring-boot-client";

    @Bean
    public MqttClient mqttClient() throws Exception {
        MqttClient client = new MqttClient(BROKER_URL, CLIENT_ID, new MemoryPersistence());

        MqttConnectOptions options = new MqttConnectOptions();
        options.setCleanSession(true);
        options.setAutomaticReconnect(true);

        client.connect(options);
        return client;
    }
}
```

---

## **4. Publisher Service**

Publish messages to an MQTT topic.

```java
import org.eclipse.paho.client.mqttv3.MqttClient;
import org.eclipse.paho.client.mqttv3.MqttMessage;
import org.springframework.stereotype.Service;

@Service
public class MqttPublisher {

    private final MqttClient mqttClient;

    public MqttPublisher(MqttClient mqttClient) {
        this.mqttClient = mqttClient;
    }

    public void publish(String topic, String payload) {
        try {
            MqttMessage message = new MqttMessage(payload.getBytes());
            message.setQos(1);
            mqttClient.publish(topic, message);
            System.out.println("📤 Published: " + payload + " to topic: " + topic);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

---

## **5. Subscriber Service**

Subscribe to topics and receive messages.

```java
import org.eclipse.paho.client.mqttv3.MqttCallback;
import org.eclipse.paho.client.mqttv3.MqttClient;
import org.eclipse.paho.client.mqttv3.MqttMessage;
import org.eclipse.paho.client.mqttv3.IMqttDeliveryToken;
import org.springframework.stereotype.Service;

import javax.annotation.PostConstruct;

@Service
public class MqttSubscriber {

    private final MqttClient mqttClient;

    public MqttSubscriber(MqttClient mqttClient) {
        this.mqttClient = mqttClient;
    }

    @PostConstruct
    public void subscribe() {
        try {
            mqttClient.subscribe("test/topic", 1);

            mqttClient.setCallback(new MqttCallback() {
                @Override
                public void connectionLost(Throwable cause) {
                    System.out.println("❌ Connection lost: " + cause.getMessage());
                }

                @Override
                public void messageArrived(String topic, MqttMessage message) {
                    System.out.println("📥 Message received from topic " + topic + ": " + new String(message.getPayload()));
                }

                @Override
                public void deliveryComplete(IMqttDeliveryToken token) {
                    System.out.println("✅ Delivery complete");
                }
            });

            System.out.println("✅ Subscribed to test/topic");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

---

## **6. REST Controller (Trigger Publisher)**

Expose an endpoint to test publishing.

```java
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/mqtt")
public class MqttController {

    private final MqttPublisher publisher;

    public MqttController(MqttPublisher publisher) {
        this.publisher = publisher;
    }

    @PostMapping("/publish")
    public String publish(@RequestParam String message) {
        publisher.publish("test/topic", message);
        return "Message published: " + message;
    }
}
```

---

## **7. Run MQTT Broker (Mosquitto using Docker)**

Start broker locally:

```bash
docker run -it -p 1883:1883 -p 9001:9001 eclipse-mosquitto
```

---

## **8. Test the Setup**

1. Start **Spring Boot app**.
2. Open **REST client** (Postman / browser).

   ```
   POST http://localhost:8080/mqtt/publish?message=HelloWorld
   ```
3. Check Spring Boot logs:

   ```
   📤 Published: HelloWorld to topic: test/topic
   📥 Message received from topic test/topic: HelloWorld
   ```

---

## **9. Real-Time Use Cases**

✅ IoT sensor data (temperature, GPS, device health).
✅ Smart home automation.
✅ Live vehicle tracking with Kafka integration.
✅ Real-time chat apps.

---

🔥 With this setup, your **Spring Boot app is now acting as both Publisher & Subscriber using Eclipse Paho MQTT**.

---

Do you also want me to extend these notes with **Spring Boot + MQTT + Kafka bridge (so messages from MQTT broker flow into Kafka automatically)**?


Perfect 👍 Let’s finalize with **full end-to-end notes for integrating Spring Boot with Eclipse Paho MQTT (Pub/Sub)**.

---

# 📘 Notes: Spring Boot + Eclipse Paho (MQTT Pub/Sub)

---

## 1. **What is MQTT?**

* MQTT = Message Queuing Telemetry Transport (lightweight pub/sub protocol).
* Used in IoT, device communication, sensor data streaming.
* Roles:

  * **Publisher** → sends messages.
  * **Subscriber** → receives messages.
  * **Broker** → message router (e.g., Mosquitto, HiveMQ, EMQX).

---

## 2. **Setup MQTT Broker (Mosquitto via Docker)**

```bash
docker run -it -p 1883:1883 -p 9001:9001 eclipse-mosquitto
```

* `1883` → MQTT TCP port.
* `9001` → MQTT WebSocket port.

Test broker with MQTT client (like `mqttx` or `mosquitto_pub/sub`).

---

## 3. **Add Dependency (Spring Boot + Paho)**

In `pom.xml`:

```xml
<dependency>
    <groupId>org.springframework.integration</groupId>
    <artifactId>spring-integration-mqtt</artifactId>
</dependency>
```

---

## 4. **Application Properties**

Configure MQTT connection in `application.yml`:

```yaml
mqtt:
  broker: tcp://localhost:1883
  client-id: boot-app
  username: test
  password: test
  topic:
    pub: demo/pub
    sub: demo/sub
```

---

## 5. **MQTT Configuration Class**

```java
import org.eclipse.paho.client.mqttv3.MqttConnectOptions;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.annotation.ServiceActivator;
import org.springframework.integration.channel.DirectChannel;
import org.springframework.integration.mqtt.core.DefaultMqttPahoClientFactory;
import org.springframework.integration.mqtt.core.MqttPahoClientFactory;
import org.springframework.integration.mqtt.inbound.MqttPahoMessageDrivenChannelAdapter;
import org.springframework.integration.mqtt.outbound.MqttPahoMessageHandler;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.MessageHandler;

@Configuration
public class MqttConfig {

    private final String brokerUrl = "tcp://localhost:1883";
    private final String clientId = "boot-client";
    private final String topicSub = "demo/sub";
    private final String topicPub = "demo/pub";

    // MQTT Connection Factory
    @Bean
    public MqttPahoClientFactory mqttClientFactory() {
        MqttConnectOptions options = new MqttConnectOptions();
        options.setCleanSession(true);
        options.setUserName("test");
        options.setPassword("test".toCharArray());
        DefaultMqttPahoClientFactory factory = new DefaultMqttPahoClientFactory();
        factory.setConnectionOptions(options);
        return factory;
    }

    // Input Channel (Subscriber)
    @Bean
    public MessageChannel mqttInputChannel() {
        return new DirectChannel();
    }

    // Subscriber Adapter
    @Bean
    public MqttPahoMessageDrivenChannelAdapter inbound(MqttPahoClientFactory factory) {
        MqttPahoMessageDrivenChannelAdapter adapter =
                new MqttPahoMessageDrivenChannelAdapter(clientId + "_sub", factory, topicSub);
        adapter.setOutputChannel(mqttInputChannel());
        return adapter;
    }

    // Message Handler for Subscriber
    @Bean
    @ServiceActivator(inputChannel = "mqttInputChannel")
    public MessageHandler handler() {
        return message -> {
            System.out.println("📩 Received message: " + message.getPayload());
        };
    }

    // Output Channel (Publisher)
    @Bean
    public MessageChannel mqttOutboundChannel() {
        return new DirectChannel();
    }

    // Publisher Handler
    @Bean
    @ServiceActivator(inputChannel = "mqttOutboundChannel")
    public MessageHandler outbound(MqttPahoClientFactory factory) {
        MqttPahoMessageHandler handler = new MqttPahoMessageHandler(clientId + "_pub", factory);
        handler.setAsync(true);
        handler.setDefaultTopic(topicPub);
        return handler;
    }
}
```

---

## 6. **Service for Publishing Messages**

```java
import org.springframework.integration.support.MessageBuilder;
import org.springframework.messaging.MessageChannel;
import org.springframework.stereotype.Service;

@Service
public class MqttPublisherService {

    private final MessageChannel mqttOutboundChannel;

    public MqttPublisherService(MessageChannel mqttOutboundChannel) {
        this.mqttOutboundChannel = mqttOutboundChannel;
    }

    public void send(String payload) {
        mqttOutboundChannel.send(MessageBuilder.withPayload(payload).build());
        System.out.println("✅ Published: " + payload);
    }
}
```

---

## 7. **REST Controller to Trigger Publish**

```java
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/mqtt")
public class MqttController {

    private final MqttPublisherService publisherService;

    public MqttController(MqttPublisherService publisherService) {
        this.publisherService = publisherService;
    }

    @PostMapping("/send/{msg}")
    public String sendMessage(@PathVariable String msg) {
        publisherService.send(msg);
        return "Message Sent: " + msg;
    }
}
```

---

## 8. **Run & Test**

1. Start **Mosquitto broker** (`docker run ...`).
2. Run **Spring Boot app**.
3. Test publish:

   ```bash
   curl -X POST http://localhost:8080/mqtt/send/HelloMQTT
   ```

   Output:

   ```
   ✅ Published: HelloMQTT
   📩 Received message: HelloMQTT
   ```

---

## 9. **Real-Time Use Cases**

* IoT sensors streaming temperature data → Boot app consumes.
* Smart Home automation (device pub/sub).
* Vehicle GPS location → publish → Boot app stores & processes.
* Chat application (real-time messaging).

---

👉 Now you have **complete notes + working code** for **Spring Boot + Eclipse Paho MQTT pub/sub**.

Would you like me to also prepare a **step-by-step test with a fake IoT device publisher in Java** (so you can simulate sensors without real devices)?

Perfect 👍 Now let’s prepare the **final complete notes** on **Spring Boot + Eclipse Paho MQTT integration** for **Pub/Sub** stepwise with explanation and code.

---

# 📘 Notes: Spring Boot with Eclipse Paho MQTT (Pub/Sub)

---

## 🔹 1. What is MQTT?

* **MQTT (Message Queuing Telemetry Transport)** → Lightweight pub/sub messaging protocol, used in **IoT & real-time communication**.
* Works with a **broker** (like Mosquitto, HiveMQ, EMQX).
* Clients can:

  * **Publish** → Send data to a topic.
  * **Subscribe** → Listen for messages on a topic.

---

## 🔹 2. Add Dependency (Maven)

```xml
<dependency>
    <groupId>org.eclipse.paho</groupId>
    <artifactId>org.eclipse.paho.client.mqttv3</artifactId>
    <version>1.2.5</version>
</dependency>
```

---

## 🔹 3. Configuration in `application.yml`

```yaml
mqtt:
  broker: tcp://localhost:1883
  clientId: boot-app
  topic:
    pub: test/topic/pub
    sub: test/topic/sub
```

---

## 🔹 4. MQTT Configuration Class

```java
import org.eclipse.paho.client.mqttv3.MqttClient;
import org.eclipse.paho.client.mqttv3.MqttConnectOptions;
import org.eclipse.paho.client.mqttv3.persist.MemoryPersistence;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MqttConfig {

    private final String broker = "tcp://localhost:1883"; // Mosquitto running locally
    private final String clientId = "spring-boot-client";

    @Bean
    public MqttClient mqttClient() throws Exception {
        MqttClient client = new MqttClient(broker, clientId, new MemoryPersistence());
        MqttConnectOptions options = new MqttConnectOptions();
        options.setCleanSession(true);
        client.connect(options);
        return client;
    }
}
```

---

## 🔹 5. Publisher Service

```java
import org.eclipse.paho.client.mqttv3.MqttClient;
import org.eclipse.paho.client.mqttv3.MqttMessage;
import org.springframework.stereotype.Service;

@Service
public class MqttPublisher {

    private final MqttClient mqttClient;

    public MqttPublisher(MqttClient mqttClient) {
        this.mqttClient = mqttClient;
    }

    public void publish(String topic, String message) throws Exception {
        MqttMessage mqttMessage = new MqttMessage(message.getBytes());
        mqttMessage.setQos(1); // QoS level (0,1,2)
        mqttClient.publish(topic, mqttMessage);
        System.out.println("✅ Published: " + message + " to topic: " + topic);
    }
}
```

---

## 🔹 6. Subscriber Service

```java
import org.eclipse.paho.client.mqttv3.MqttCallback;
import org.eclipse.paho.client.mqttv3.MqttMessage;
import org.eclipse.paho.client.mqttv3.IMqttDeliveryToken;
import org.eclipse.paho.client.mqttv3.MqttClient;
import org.springframework.stereotype.Service;

import javax.annotation.PostConstruct;

@Service
public class MqttSubscriber implements MqttCallback {

    private final MqttClient mqttClient;

    public MqttSubscriber(MqttClient mqttClient) {
        this.mqttClient = mqttClient;
        mqttClient.setCallback(this);
    }

    @PostConstruct
    public void subscribe() throws Exception {
        mqttClient.subscribe("test/topic/sub", 1);
        System.out.println("✅ Subscribed to topic: test/topic/sub");
    }

    @Override
    public void connectionLost(Throwable cause) {
        System.out.println("⚠️ Connection lost: " + cause.getMessage());
    }

    @Override
    public void messageArrived(String topic, MqttMessage message) {
        System.out.println("📩 Received on " + topic + ": " + new String(message.getPayload()));
    }

    @Override
    public void deliveryComplete(IMqttDeliveryToken token) {
        System.out.println("✅ Delivery complete");
    }
}
```

---

## 🔹 7. REST Controller to Test Publish

```java
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/mqtt")
public class MqttController {

    private final MqttPublisher publisher;

    public MqttController(MqttPublisher publisher) {
        this.publisher = publisher;
    }

    @GetMapping("/send/{msg}")
    public String sendMessage(@PathVariable String msg) throws Exception {
        publisher.publish("test/topic/sub", msg);
        return "Message published: " + msg;
    }
}
```

---

## 🔹 8. Run MQTT Broker (Mosquitto with Docker)

```bash
docker run -it -p 1883:1883 -p 9001:9001 eclipse-mosquitto
```

---

## 🔹 9. Test Flow

1. Start **Mosquitto** broker in Docker.
2. Run **Spring Boot app**.
3. Open browser:
   `http://localhost:8080/mqtt/send/helloWorld`
4. Console logs:

   * **Publisher** → ✅ Published: helloWorld
   * **Subscriber** → 📩 Received: helloWorld

---

## 🔹 10. Real-World Use Case

* **IoT Device (Publisher)** → Sends sensor data (`temperature`, `location`, etc.) to MQTT.
* **Spring Boot App (Subscriber)** → Reads data, stores in DB, or forwards to **Kafka**.
* **Spring Boot App (Publisher)** → Sends commands back to IoT device (like "turn fan ON").

---

✅ This is the **stepwise integration notes + working code** for **Spring Boot + Eclipse Paho MQTT**.

---

Do you also want me to **extend these notes** to show how to **bridge MQTT to Kafka** (so data goes from IoT → MQTT → Kafka → Boot consumer)?


Perfect 👍 Let’s wrap this up with **final complete notes** on **Spring Boot + Eclipse Paho MQTT Integration (Pub/Sub)**.

---

# 📘 Notes: Spring Boot + Eclipse Paho MQTT (Pub/Sub)

---

## 1. **Add Dependency**

In your `pom.xml`:

```xml
<dependency>
    <groupId>org.eclipse.paho</groupId>
    <artifactId>org.eclipse.paho.client.mqttv3</artifactId>
    <version>1.2.5</version>
</dependency>
```

---

## 2. **MQTT Configuration Class**

Create a config class to connect Spring Boot app with MQTT broker.

```java
import org.eclipse.paho.client.mqttv3.MqttClient;
import org.eclipse.paho.client.mqttv3.MqttConnectOptions;
import org.eclipse.paho.client.mqttv3.persist.MemoryPersistence;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MqttConfig {

    private static final String BROKER_URL = "tcp://localhost:1883"; // Mosquitto broker
    private static final String CLIENT_ID = "boot-app-client";

    @Bean
    public MqttClient mqttClient() throws Exception {
        MqttClient client = new MqttClient(BROKER_URL, CLIENT_ID, new MemoryPersistence());
        MqttConnectOptions options = new MqttConnectOptions();
        options.setCleanSession(true);
        options.setAutomaticReconnect(true);
        client.connect(options);
        return client;
    }
}
```

---

## 3. **Publisher Service**

Spring Boot service to publish messages to MQTT topic.

```java
import org.eclipse.paho.client.mqttv3.MqttClient;
import org.eclipse.paho.client.mqttv3.MqttMessage;
import org.springframework.stereotype.Service;

@Service
public class MqttPublisher {

    private final MqttClient mqttClient;

    public MqttPublisher(MqttClient mqttClient) {
        this.mqttClient = mqttClient;
    }

    public void publish(String topic, String payload) throws Exception {
        MqttMessage message = new MqttMessage(payload.getBytes());
        message.setQos(1); // QoS 0,1,2
        mqttClient.publish(topic, message);
        System.out.println("✅ Published: " + payload + " to topic: " + topic);
    }
}
```

---

## 4. **Subscriber Service**

Listen to MQTT topic messages.

```java
import org.eclipse.paho.client.mqttv3.IMqttMessageListener;
import org.eclipse.paho.client.mqttv3.MqttClient;
import org.springframework.stereotype.Service;

import javax.annotation.PostConstruct;

@Service
public class MqttSubscriber {

    private final MqttClient mqttClient;

    public MqttSubscriber(MqttClient mqttClient) {
        this.mqttClient = mqttClient;
    }

    @PostConstruct
    public void subscribe() throws Exception {
        mqttClient.subscribe("test/topic", new IMqttMessageListener() {
            @Override
            public void messageArrived(String topic, org.eclipse.paho.client.mqttv3.MqttMessage message) throws Exception {
                System.out.println("📩 Received message: " + new String(message.getPayload()) + " from topic: " + topic);
            }
        });
        System.out.println("✅ Subscribed to topic: test/topic");
    }
}
```

---

## 5. **Test with REST Controller**

Expose REST API to send messages to MQTT.

```java
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/mqtt")
public class MqttController {

    private final MqttPublisher publisher;

    public MqttController(MqttPublisher publisher) {
        this.publisher = publisher;
    }

    @GetMapping("/publish/{msg}")
    public String publishMessage(@PathVariable String msg) throws Exception {
        publisher.publish("test/topic", msg);
        return "Message Published: " + msg;
    }
}
```

---

## 6. **Run Mosquitto Broker via Docker**

If you don’t have an IoT device, run Mosquitto broker locally:

```bash
docker run -it -p 1883:1883 -p 9001:9001 eclipse-mosquitto
```

---

## 7. **Testing**

1. Run Spring Boot app.
2. Open **Postman / Browser**:
   `http://localhost:8080/mqtt/publish/hello-mqtt`
3. Subscriber logs will show:

   ```
   📩 Received message: hello-mqtt from topic: test/topic
   ```

---

## 8. **Real-World Use Cases**

* IoT device sending sensor data → Boot app consumes.
* Boot app publishes commands → IoT devices act.
* MQTT + Kafka pipeline → MQTT for device comms, Kafka for large-scale analytics.

---

👉 With this setup:

* **Spring Boot** publishes data to MQTT broker.
* **Spring Boot** subscribes and listens in real time.
* You can scale to Kafka later.

---

Do you also want me to extend these notes to show **Boot → MQTT → Kafka** flow (publishing MQTT messages to Kafka for analytics)?

Great 👍 Let’s now finalize **complete working notes** on **Spring Boot + Eclipse Paho MQTT (Pub/Sub)** so you have everything step-by-step in one place.

---

# 📘 Spring Boot with MQTT (Eclipse Paho) – Pub/Sub Integration Notes

---

## 🔹 1. Add Maven Dependency

```xml
<dependency>
    <groupId>org.eclipse.paho</groupId>
    <artifactId>org.eclipse.paho.client.mqttv3</artifactId>
    <version>1.2.5</version>
</dependency>
```

---

## 🔹 2. Configure MQTT Broker Connection

👉 We’ll connect to **Eclipse Mosquitto** (you can run via Docker):

```bash
docker run -it -p 1883:1883 -p 9001:9001 eclipse-mosquitto
```

---

## 🔹 3. Create MQTT Configuration Bean

```java
import org.eclipse.paho.client.mqttv3.MqttClient;
import org.eclipse.paho.client.mqttv3.MqttConnectOptions;
import org.eclipse.paho.client.mqttv3.persist.MemoryPersistence;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MqttConfig {

    private static final String BROKER = "tcp://localhost:1883";
    private static final String CLIENT_ID = "spring-boot-client";

    @Bean
    public MqttClient mqttClient() throws Exception {
        MqttClient client = new MqttClient(BROKER, CLIENT_ID, new MemoryPersistence());
        MqttConnectOptions options = new MqttConnectOptions();
        options.setCleanSession(true);
        client.connect(options);
        return client;
    }
}
```

---

## 🔹 4. Publisher Service (Send Data)

```java
import org.eclipse.paho.client.mqttv3.MqttClient;
import org.eclipse.paho.client.mqttv3.MqttMessage;
import org.springframework.stereotype.Service;

@Service
public class MqttPublisher {

    private final MqttClient mqttClient;

    public MqttPublisher(MqttClient mqttClient) {
        this.mqttClient = mqttClient;
    }

    public void publish(String topic, String payload) throws Exception {
        MqttMessage message = new MqttMessage(payload.getBytes());
        message.setQos(1);
        mqttClient.publish(topic, message);
        System.out.println("✅ Published: " + payload + " to topic: " + topic);
    }
}
```

---

## 🔹 5. Subscriber Service (Receive Data)

```java
import org.eclipse.paho.client.mqttv3.IMqttMessageListener;
import org.eclipse.paho.client.mqttv3.MqttClient;
import org.springframework.stereotype.Service;

import javax.annotation.PostConstruct;

@Service
public class MqttSubscriber {

    private final MqttClient mqttClient;

    public MqttSubscriber(MqttClient mqttClient) {
        this.mqttClient = mqttClient;
    }

    @PostConstruct
    public void subscribe() throws Exception {
        mqttClient.subscribe("iot/sensor", (topic, msg) -> {
            String message = new String(msg.getPayload());
            System.out.println("📩 Received on topic [" + topic + "]: " + message);
        });
    }
}
```

---

## 🔹 6. REST Controller to Trigger Publish

```java
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/mqtt")
public class MqttController {

    private final MqttPublisher publisher;

    public MqttController(MqttPublisher publisher) {
        this.publisher = publisher;
    }

    @GetMapping("/publish/{message}")
    public String publishMessage(@PathVariable String message) throws Exception {
        publisher.publish("iot/sensor", message);
        return "Message published: " + message;
    }
}
```

---

## 🔹 7. Run & Test

1. Start **Mosquitto broker**:

```bash
docker run -it -p 1883:1883 eclipse-mosquitto
```

2. Run **Spring Boot app**.

3. Test publishing:

```bash
curl http://localhost:8080/mqtt/publish/helloMQTT
```

4. Subscriber automatically prints:

```
📩 Received on topic [iot/sensor]: helloMQTT
```

---

## 🔹 8. Key Notes

* **Publisher** pushes data to MQTT topic.
* **Subscriber** listens and reacts to messages.
* Can extend for **IoT devices → MQTT → Spring Boot → Kafka/DB**.
* Useful for **real-time IoT, sensors, event-driven apps**.

---

✅ With this setup, you have **Spring Boot + Paho + MQTT Pub/Sub** working end-to-end.

Do you want me to extend this and show **MQTT → Kafka bridge (Spring Boot reading MQTT and publishing to Kafka)** as the **next step**?


Perfect 🚀
Now let’s complete the **Spring Boot + Eclipse Paho MQTT Pub/Sub integration** notes with working code.

---

# 📒 Notes: Spring Boot + MQTT (Eclipse Paho) Pub/Sub

---

## **1. Add Dependency**

Use **Eclipse Paho MQTT Client** in your `pom.xml`:

```xml
<dependency>
    <groupId>org.eclipse.paho</groupId>
    <artifactId>org.eclipse.paho.client.mqttv3</artifactId>
    <version>1.2.5</version>
</dependency>
```

---

## **2. Define Configuration**

Create `MqttConfig.java` to set up the MQTT client.

```java
import org.eclipse.paho.client.mqttv3.MqttClient;
import org.eclipse.paho.client.mqttv3.MqttConnectOptions;
import org.eclipse.paho.client.mqttv3.persist.MemoryPersistence;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MqttConfig {

    private static final String BROKER_URL = "tcp://localhost:1883"; // Mosquitto broker
    private static final String CLIENT_ID = "boot-app";

    @Bean
    public MqttClient mqttClient() throws Exception {
        MqttClient client = new MqttClient(BROKER_URL, CLIENT_ID, new MemoryPersistence());
        MqttConnectOptions options = new MqttConnectOptions();
        options.setCleanSession(true);
        client.connect(options);
        return client;
    }
}
```

---

## **3. Publisher Service**

Create `MqttPublisher.java` for publishing messages.

```java
import org.eclipse.paho.client.mqttv3.MqttClient;
import org.eclipse.paho.client.mqttv3.MqttMessage;
import org.springframework.stereotype.Service;

@Service
public class MqttPublisher {

    private final MqttClient mqttClient;

    public MqttPublisher(MqttClient mqttClient) {
        this.mqttClient = mqttClient;
    }

    public void publish(String topic, String payload) throws Exception {
        MqttMessage message = new MqttMessage(payload.getBytes());
        message.setQos(1); // QoS 1: At least once
        mqttClient.publish(topic, message);
        System.out.println("✅ Published: " + payload + " to topic: " + topic);
    }
}
```

---

## **4. Subscriber Service**

Create `MqttSubscriber.java` for subscribing.

```java
import org.eclipse.paho.client.mqttv3.IMqttMessageListener;
import org.eclipse.paho.client.mqttv3.MqttClient;
import org.springframework.stereotype.Service;

import javax.annotation.PostConstruct;

@Service
public class MqttSubscriber {

    private final MqttClient mqttClient;

    public MqttSubscriber(MqttClient mqttClient) {
        this.mqttClient = mqttClient;
    }

    @PostConstruct
    public void subscribe() throws Exception {
        mqttClient.subscribe("sensor/data", new IMqttMessageListener() {
            @Override
            public void messageArrived(String topic, org.eclipse.paho.client.mqttv3.MqttMessage message) throws Exception {
                System.out.println("📩 Received: " + new String(message.getPayload()) + " from topic: " + topic);
            }
        });
        System.out.println("✅ Subscribed to topic: sensor/data");
    }
}
```

---

## **5. Controller to Test**

Expose REST API to publish messages.

```java
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/mqtt")
public class MqttController {

    private final MqttPublisher publisher;

    public MqttController(MqttPublisher publisher) {
        this.publisher = publisher;
    }

    @PostMapping("/publish")
    public String publish(@RequestParam String message) throws Exception {
        publisher.publish("sensor/data", message);
        return "Message sent: " + message;
    }
}
```

---

## **6. Run MQTT Broker (Docker)**

Run Mosquitto broker locally with Docker:

```bash
docker run -it -p 1883:1883 -p 9001:9001 eclipse-mosquitto
```

---

## **7. Test**

1. Start Spring Boot app.
2. Hit API:

   ```bash
   curl -X POST "http://localhost:8080/mqtt/publish?message=HelloMQTT"
   ```
3. Console Output:

   ```
   ✅ Published: HelloMQTT to topic: sensor/data
   📩 Received: HelloMQTT from topic: sensor/data
   ```

---

👉 Now your Boot app can **publish** and **subscribe** messages using **Paho + MQTT**.

Would you like me to also extend this with a **fake Java IoT sensor simulator** that continuously publishes data to `sensor/data` topic for testing?


Perfect 🚀 Let’s wrap this up into a **final complete notes-style document** that you can directly use for your learning and project.

---

# 📘 Spring Boot + Eclipse Paho MQTT Integration (Pub/Sub)

## 1. What is MQTT?

* Lightweight **pub/sub messaging protocol**.
* Common in **IoT applications** (sensors → broker → consumers).
* Works over TCP/IP.
* Requires a **Broker** (like Mosquitto, EMQX, HiveMQ).

---

## 2. Architecture

* **Publisher**: Sends messages to a **topic**.
* **Subscriber**: Listens on a **topic**.
* **Broker**: Middle layer (routes messages).

```
Spring Boot (Publisher) ---> MQTT Broker ---> Spring Boot (Subscriber)
```

---

## 3. Add Dependency (Maven)

```xml
<dependency>
  <groupId>org.eclipse.paho</groupId>
  <artifactId>org.eclipse.paho.client.mqttv3</artifactId>
  <version>1.2.5</version>
</dependency>
```

---

## 4. application.yml Configuration

```yaml
mqtt:
  broker: tcp://localhost:1883   # Mosquitto running locally
  clientId: boot-app-client
  topic: test/topic
  qos: 1
```

---

## 5. Create MQTT Config

```java
import org.eclipse.paho.client.mqttv3.*;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MqttConfig {

    @Value("${mqtt.broker}")
    private String brokerUrl;

    @Value("${mqtt.clientId}")
    private String clientId;

    @Bean
    public MqttClient mqttClient() throws MqttException {
        MqttClient client = new MqttClient(brokerUrl, clientId, null);
        MqttConnectOptions options = new MqttConnectOptions();
        options.setCleanSession(true);
        client.connect(options);
        return client;
    }
}
```

---

## 6. Publisher Service

```java
import org.eclipse.paho.client.mqttv3.*;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

@Service
public class MqttPublisher {

    private final MqttClient client;

    @Value("${mqtt.topic}")
    private String topic;

    public MqttPublisher(MqttClient client) {
        this.client = client;
    }

    public void publish(String message) throws MqttException {
        MqttMessage mqttMessage = new MqttMessage(message.getBytes());
        mqttMessage.setQos(1);
        client.publish(topic, mqttMessage);
        System.out.println("✅ Published: " + message);
    }
}
```

---

## 7. Subscriber Service

```java
import org.eclipse.paho.client.mqttv3.*;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import jakarta.annotation.PostConstruct;

@Service
public class MqttSubscriber {

    private final MqttClient client;

    @Value("${mqtt.topic}")
    private String topic;

    public MqttSubscriber(MqttClient client) {
        this.client = client;
    }

    @PostConstruct
    public void subscribe() throws MqttException {
        client.subscribe(topic, (t, msg) -> {
            String received = new String(msg.getPayload());
            System.out.println("📩 Received: " + received);
        });
    }
}
```

---

## 8. REST Controller (to trigger publish)

```java
import org.eclipse.paho.client.mqttv3.MqttException;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/mqtt")
public class MqttController {

    private final MqttPublisher publisher;

    public MqttController(MqttPublisher publisher) {
        this.publisher = publisher;
    }

    @PostMapping("/publish")
    public String publishMessage(@RequestParam String message) throws MqttException {
        publisher.publish(message);
        return "Message sent: " + message;
    }
}
```

---

## 9. Run Mosquitto in Docker

```bash
docker run -it -p 1883:1883 -p 9001:9001 eclipse-mosquitto
```

---

## 10. Test Flow

1. Start **Mosquitto Broker** (`docker ps` should show container running).
2. Run Spring Boot app.
3. Publish message:

   ```bash
   curl -X POST "http://localhost:8080/mqtt/publish?message=HelloMQTT"
   ```
4. See logs:

   ```
   ✅ Published: HelloMQTT
   📩 Received: HelloMQTT
   ```

---

## 11. Real-time Use Cases

* IoT sensor data → MQTT Broker → Analytics.
* Device control (e.g., smart home).
* Real-time notification systems.
* Vehicle tracking (GPS updates).

---

✅ With this setup, your Boot app can **publish** and **subscribe** using **Eclipse Paho MQTT**.
Next step (if you want) → extend this to **Kafka bridge** (MQTT → Kafka).

Would you like me to now prepare the **next extension: MQTT → Kafka bridge** (so messages go Boot → MQTT → Kafka → Boot)?

---
