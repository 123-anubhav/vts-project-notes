Got it ✅ You want **end-to-end notes** on how to do **testing and troubleshooting in Kafka**, specifically for a **microservice-based Kafka Streams app**.
I’ll prepare this as **stepwise notes** with practical examples and troubleshooting strategies.

---

# 📒 Kafka Testing & Troubleshooting Notes for Microservice + Kafka Streams App

---

## 1. **Types of Kafka Testing**

In a microservice + Kafka Streams setup, testing happens at multiple levels:

1. **Unit Testing**

   * Test your producer/consumer logic in isolation.
   * Mock Kafka dependencies.
   * Example: check that messages are serialized properly.

2. **Integration Testing**

   * Test with embedded Kafka cluster (`EmbeddedKafkaBroker` or `Testcontainers`).
   * Validate producer sends, consumer receives, and Streams process correctly.

3. **End-to-End Testing**

   * Run microservices + Kafka cluster together.
   * Push sample input → observe output topic/db → validate result.

4. **Performance Testing**

   * Use **Apache JMeter** or **kafka-producer-perf-test.sh / kafka-consumer-perf-test.sh**.
   * Validate latency, throughput, and partition distribution.

---

## 2. **Testing Approaches for Kafka Streams**

### a) **Unit Test with TopologyTestDriver**

Kafka Streams has an inbuilt `TopologyTestDriver` for unit testing.

🔹 Example:

```java
@Test
void testStreamProcessing() {
    Properties props = new Properties();
    props.put(StreamsConfig.APPLICATION_ID_CONFIG, "test-app");
    props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "dummy:1234");

    StreamsBuilder builder = new StreamsBuilder();
    KStream<String, String> input = builder.stream("input-topic");
    input.mapValues(value -> value.toUpperCase())
         .to("output-topic");

    TopologyTestDriver testDriver = new TopologyTestDriver(builder.build(), props);

    TestInputTopic<String, String> inputTopic =
        testDriver.createInputTopic("input-topic", new StringSerializer(), new StringSerializer());
    TestOutputTopic<String, String> outputTopic =
        testDriver.createOutputTopic("output-topic", new StringDeserializer(), new StringDeserializer());

    inputTopic.pipeInput("key1", "hello");
    assertEquals("HELLO", outputTopic.readValue());
}
```

👉 Use when testing **transformations** in Kafka Streams without real Kafka.

---

### b) **Integration Test with Embedded Kafka**

Spring Kafka provides `@EmbeddedKafka`.

🔹 Example:

```java
@SpringBootTest
@EmbeddedKafka(partitions = 1, topics = { "input-topic", "output-topic" })
class KafkaIntegrationTest {

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    @Autowired
    private ConsumerFactory<String, String> consumerFactory;

    @Test
    void testMessageFlow() {
        kafkaTemplate.send("input-topic", "test-message");

        Consumer<String, String> consumer = consumerFactory.createConsumer();
        consumer.subscribe(Collections.singleton("output-topic"));
        ConsumerRecords<String, String> records = KafkaTestUtils.getRecords(consumer);

        assertFalse(records.isEmpty());
    }
}
```

👉 Use when validating **producer → stream → consumer** flow in microservice.

---

### c) **End-to-End Test with Testcontainers**

Testcontainers spins up Kafka in Docker during tests.

🔹 Example:

```java
@Testcontainers
class KafkaE2ETest {
    @Container
    public static KafkaContainer kafka = new KafkaContainer("confluentinc/cp-kafka:7.4.0");

    @Test
    void testKafkaEndToEnd() {
        String bootstrapServers = kafka.getBootstrapServers();
        // produce + consume to verify
    }
}
```

👉 Best for **CI/CD pipelines**.

---

## 3. **Troubleshooting in Kafka Microservices**

### a) **Common Issues & Fixes**

1. **Message not consumed**

   * Check **consumer group ID** (each microservice must have unique group id unless sharing load).
   * Ensure **auto.offset.reset=earliest** for first run.
   * Verify topic exists and has correct partitions.

2. **Serialization / Deserialization Errors**

   * Add logging around SerDes.
   * Use custom Serdes if working with POJOs.
   * Validate schemas with Avro/JSON.

3. **Lag in Consumers**

   * Run:

     ```bash
     kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group <group-name>
     ```
   * Tune:

     * `fetch.min.bytes`
     * `max.poll.records`
     * Scale consumers.

4. **Duplicate Messages**

   * Enable **idempotent producer**:

     ```properties
     enable.idempotence=true
     ```
   * Use `ExactlyOnce` processing in Kafka Streams.

5. **Stream Application Crashes**

   * Use `DefaultProductionExceptionHandler` & `DefaultDeserializationExceptionHandler`.
   * Restart strategy with Spring Boot + Kubernetes.

---

### b) **Debugging Tools**

* **Kafka CLI**

  * `kafka-topics.sh --list`
  * `kafka-console-producer.sh --topic test --bootstrap-server localhost:9092`
  * `kafka-console-consumer.sh --topic test --from-beginning`

* **Kafka UI Tools**

  * Conduktor, Kafdrop, Redpanda Console.
  * Helps visualize topics, partitions, offsets.

* **Metrics**

  * Expose via JMX → Grafana/Prometheus.
  * Check processing rate, lag, rebalance time.

---

### c) **Microservice Troubleshooting Checklist**

1. **Is service registered with Eureka/Service Registry (if using)?**
2. **Are bootstrap servers correct?**
3. **Is topic auto-created or manually created?**
4. **Is partition count sufficient for scaling?**
5. **Check logs:**

   * `SerializationException`
   * `WakeupException`
   * `TimeoutException`

---

## 4. **Best Practices**

* Use **DLQ (Dead Letter Queue)** for bad messages.
* Implement **retry mechanism** with exponential backoff.
* Use **schema registry** (Avro/Protobuf).
* Keep test topics separate from production.
* Automate tests in **CI/CD pipeline** with Testcontainers.

---

✅ This gives you **full notes**: testing (unit → integration → E2E), troubleshooting (common errors + fixes), and best practices for Kafka Streams microservices.

Would you like me to also prepare a **real-world test strategy document** (like a checklist/plan for your vehicle tracking project in Kafka)?


Perfect 🚀
Let’s prepare a **Real-World Kafka Test Strategy Document** tailored for a **vehicle tracking system** using **Kafka + Kafka Streams + Microservices**.

---

# 📘 Kafka Test Strategy for Vehicle Tracking System

---

## 1. **System Overview**

* Vehicles send location & telemetry → Kafka Topic (`vehicle-location-events`)
* Kafka Streams process data (filter invalid, enrich with geo info, detect events)
* Downstream microservices consume processed data (UI, alerts, DB persistence)
* Use **Postgres + Hibernate Spatial** for storage
* Use **OpenCV** / AI service for video analytics events (optional microservice)

---

## 2. **Kafka Topics**

* `vehicle-location-events` → raw GPS events
* `vehicle-validated-events` → after validation/enrichment
* `vehicle-alerts` → abnormal events (overspeeding, deviation)
* `vehicle-dashboard` → for UI dashboard consumption

---

## 3. **Test Types & Plan**

### 🔹 **A. Unit Testing**

**Goal:** Validate transformation & filtering logic in Kafka Streams.

✅ Tools:

* JUnit + TopologyTestDriver
* Mockito for services

✅ Scenarios:

* Transform raw GPS → Enriched GPS with geo-boundary
* Drop invalid lat/long
* Convert speed unit (m/s → km/h)

---

### 🔹 **B. Integration Testing**

**Goal:** Validate end-to-end flow within a single microservice.

✅ Tools:

* Spring Kafka + `@EmbeddedKafka`
* KafkaTestUtils

✅ Scenarios:

* Produce a raw GPS event → check processed output in `vehicle-validated-events`
* Validate Kafka SerDe (JSON/Avro) works
* Ensure DLQ receives malformed messages

---

### 🔹 **C. End-to-End Testing**

**Goal:** Validate complete flow across microservices.

✅ Tools:

* Testcontainers (Kafka + Postgres) in CI/CD
* Kafka CLI for manual testing

✅ Scenarios:

1. Produce a vehicle event → verify DB persistence
2. Produce an overspeeding event → alert service should consume from `vehicle-alerts`
3. Dashboard service consumes from `vehicle-dashboard` correctly

---

### 🔹 **D. Performance Testing**

**Goal:** Check latency & throughput under load.

✅ Tools:

* Kafka perf tools (`kafka-producer-perf-test.sh`)
* Apache JMeter with Kafka plugin

✅ Scenarios:

* Produce 100K events/min → system should process under 2 sec avg latency
* Consumer lag should remain < threshold (say 500 messages)
* Partition rebalancing should not cause > 5 sec downtime

---

### 🔹 **E. Chaos / Fault Tolerance Testing**

**Goal:** Validate resilience under failure.

✅ Tools:

* Chaos Monkey (Kubernetes)
* Manual broker shutdown/restart

✅ Scenarios:

* Kill one Kafka broker → ensure microservices continue consuming
* Kill consumer service → Kafka retains offsets, resumes processing later
* Network latency injection → check retries/backoff

---

## 4. **Troubleshooting Guide**

| Issue                 | Diagnosis Command                                              | Fix                                                   |
| --------------------- | -------------------------------------------------------------- | ----------------------------------------------------- |
| Messages not consumed | `kafka-consumer-groups.sh --describe --group vehicle-consumer` | Check group ID, auto.offset.reset, topic subscription |
| Serialization error   | Logs show `SerializationException`                             | Check SerDe config, schema registry, input format     |
| High consumer lag     | Monitor with JMX / Prometheus                                  | Scale consumers, increase partitions                  |
| Duplicate messages    | Logs & DB show multiple inserts                                | Enable idempotence + EOS (Exactly Once Semantics)     |
| Stream app crash      | `DefaultDeserializationExceptionHandler` log                   | Add error handler, route bad messages to DLQ          |

---

## 5. **Best Practices**

* **DLQ Setup**: `vehicle-dlq-events` for invalid messages
* **Schema Registry**: Avro/Protobuf for consistent schema
* **Monitoring**:

  * Prometheus + Grafana for Kafka lag
  * Kafka UI tool (Kafdrop / Conduktor)
* **CI/CD**:

  * Unit + Integration → run on every PR
  * End-to-End (Testcontainers) → run nightly
* **Load Test** → Run before production rollout

---

## 6. **Checklist Before Go-Live**

✅ All Kafka topics created & partitioned
✅ Monitoring dashboards in place (Lag, Throughput, Error rate)
✅ DLQ configured & tested
✅ Auto-scaling consumers tested
✅ Fault recovery tested (broker down / consumer restart)
✅ End-to-end business flows validated

---

👉 This test strategy can be directly applied to your **Vehicle Tracking Kafka App**.

Would you like me to also **write a sample CI/CD pipeline YAML (GitHub Actions or Jenkins)** to automatically run unit + integration + end-to-end Kafka tests?


Awesome 🚀 Let’s create a **sample CI/CD pipeline** that runs **unit + integration + end-to-end Kafka tests** for your **Kafka Streams microservice**.
I’ll give you **two versions**:

* **GitHub Actions (YAML)** → if you’re using GitHub repo
* **Jenkinsfile (Groovy)** → if you’re using Jenkins

---

# 🔹 1. GitHub Actions Workflow (Kafka + Microservice Testing)

Create file: `.github/workflows/kafka-ci.yml`

```yaml
name: Kafka Microservice CI

on:
  push:
    branches: [ "main", "dev" ]
  pull_request:
    branches: [ "main", "dev" ]

jobs:
  build-test:
    runs-on: ubuntu-latest

    services:
      kafka:
        image: confluentinc/cp-kafka:7.4.0
        ports:
          - 9092:9092
        env:
          KAFKA_BROKER_ID: 1
          KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
          KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
          KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      zookeeper:
        image: confluentinc/cp-zookeeper:7.4.0
        ports:
          - 2181:2181
        env:
          ZOOKEEPER_CLIENT_PORT: 2181

    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Cache Maven
        uses: actions/cache@v3
        with:
          path: ~/.m2
          key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}
          restore-keys: ${{ runner.os }}-m2

      - name: Build and Run Unit Tests
        run: mvn clean test -Punit

      - name: Run Integration Tests (EmbeddedKafka)
        run: mvn test -Pintegration

      - name: Run End-to-End Tests (Testcontainers)
        run: mvn test -Pe2e

      - name: Package Application
        run: mvn clean package -DskipTests
```

👉 This workflow does:

* Starts **Kafka + Zookeeper in services**
* Runs **unit tests → integration tests → E2E tests** (separate Maven profiles recommended)
* Packages the microservice

---

# 🔹 2. Jenkins Pipeline (Jenkinsfile)

Put this `Jenkinsfile` in your repo root:

```groovy
pipeline {
    agent any

    environment {
        MAVEN_OPTS = "-Dmaven.test.failure.ignore=false"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Unit Test') {
            steps {
                sh 'mvn clean test -Punit'
            }
        }

        stage('Integration Test (Embedded Kafka)') {
            steps {
                sh 'mvn test -Pintegration'
            }
        }

        stage('End-to-End Test (Testcontainers)') {
            steps {
                sh 'mvn test -Pe2e'
            }
        }

        stage('Package App') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }
    }

    post {
        always {
            junit '**/target/surefire-reports/*.xml'
            archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
        }
        failure {
            echo "Build failed! Check logs."
        }
    }
}
```

👉 This pipeline does:

* Runs **unit tests**
* Runs **integration tests** (with EmbeddedKafka)
* Runs **end-to-end tests** (with Testcontainers spinning Kafka + DB in Docker)
* Publishes test reports + artifacts

---

# 🔹 3. Test Profiles in `pom.xml`

You’ll need **Maven profiles** to separate test types:

```xml
<profiles>
    <profile>
        <id>unit</id>
        <properties>
            <test.includes>**/unit/**</test.includes>
        </properties>
    </profile>

    <profile>
        <id>integration</id>
        <properties>
            <test.includes>**/integration/**</test.includes>
        </properties>
    </profile>

    <profile>
        <id>e2e</id>
        <properties>
            <test.includes>**/e2e/**</test.includes>
        </properties>
    </profile>
</profiles>
```

---

✅ With this setup:

* **Unit Tests** → run fast with `TopologyTestDriver`
* **Integration Tests** → run with `EmbeddedKafka`
* **E2E Tests** → run with Testcontainers Kafka + DB
* Works in **GitHub Actions or Jenkins**

---

👉 Do you want me to also include **performance testing (Kafka load test using JMeter or Kafka-perf tools) into CI/CD pipeline**?


Perfect ✅ let’s extend the **CI/CD pipeline** with **performance testing** for Kafka.

We’ll cover:

* How to **integrate Kafka performance tests** (using Kafka’s CLI tools + JMeter)
* How to **run them in CI/CD** (GitHub Actions + Jenkins)
* How to **report results**

---

# 🔹 1. Kafka Performance Testing Tools

### A) **Kafka Built-in Tools**

* `kafka-producer-perf-test.sh`

  ```bash
  kafka-producer-perf-test.sh \
    --topic perf-test \
    --num-records 500000 \
    --record-size 100 \
    --throughput -1 \
    --producer-props bootstrap.servers=localhost:9092
  ```

* `kafka-consumer-perf-test.sh`

  ```bash
  kafka-consumer-perf-test.sh \
    --bootstrap-server localhost:9092 \
    --topic perf-test \
    --messages 500000
  ```

👉 Outputs: latency, throughput (MB/sec), records/sec

---

### B) **Apache JMeter (Kafka Plugin)**

* JMeter + **Kafka plugin** → simulate producers/consumers
* Supports ramp-up, concurrency, think-time
* Export results as CSV/HTML for reporting

---

# 🔹 2. GitHub Actions Workflow with Performance Test

Update `.github/workflows/kafka-ci.yml`:

```yaml
name: Kafka Microservice CI

on:
  push:
    branches: [ "main", "dev" ]

jobs:
  build-test:
    runs-on: ubuntu-latest

    services:
      kafka:
        image: confluentinc/cp-kafka:7.4.0
        ports:
          - 9092:9092
        env:
          KAFKA_BROKER_ID: 1
          KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
          KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
          KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      zookeeper:
        image: confluentinc/cp-zookeeper:7.4.0
        ports:
          - 2181:2181
        env:
          ZOOKEEPER_CLIENT_PORT: 2181

    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Build and Run Unit Tests
        run: mvn clean test -Punit

      - name: Run Integration Tests
        run: mvn test -Pintegration

      - name: Run End-to-End Tests
        run: mvn test -Pe2e

      - name: Kafka Producer Perf Test
        run: |
          docker exec ${{ job.services.kafka.id }} \
          kafka-producer-perf-test --topic perf-test \
          --num-records 10000 --record-size 100 \
          --throughput -1 \
          --producer-props bootstrap.servers=localhost:9092

      - name: Kafka Consumer Perf Test
        run: |
          docker exec ${{ job.services.kafka.id }} \
          kafka-consumer-perf-test --bootstrap-server localhost:9092 \
          --topic perf-test --messages 10000
```

👉 Runs perf tests **inside Kafka container** after unit/integration tests.

---

# 🔹 3. Jenkinsfile with Performance Stage

Update `Jenkinsfile`:

```groovy
pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Unit Tests') {
            steps {
                sh 'mvn clean test -Punit'
            }
        }

        stage('Integration Tests') {
            steps {
                sh 'mvn test -Pintegration'
            }
        }

        stage('E2E Tests') {
            steps {
                sh 'mvn test -Pe2e'
            }
        }

        stage('Performance Test') {
            steps {
                sh '''
                  docker exec kafka \
                  kafka-producer-perf-test --topic perf-test \
                  --num-records 100000 --record-size 200 \
                  --throughput -1 \
                  --producer-props bootstrap.servers=localhost:9092
                  
                  docker exec kafka \
                  kafka-consumer-perf-test --bootstrap-server localhost:9092 \
                  --topic perf-test --messages 100000
                '''
            }
        }

        stage('Package') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }
    }

    post {
        always {
            junit '**/target/surefire-reports/*.xml'
            archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
        }
    }
}
```

---

# 🔹 4. JMeter Integration (Optional Advanced)

If you want **JMeter performance results in CI/CD**:

1. Create `kafka-performance-test.jmx`

   * Add Kafka Producer Sampler
   * Add Kafka Consumer Sampler
   * Save results to `results.jtl`

2. Add stage in **GitHub Actions**:

   ```yaml
   - name: Run JMeter Kafka Performance Test
     run: |
       jmeter -n -t kafka-performance-test.jmx -l results.jtl -e -o results-html
   - name: Upload JMeter Report
     uses: actions/upload-artifact@v3
     with:
       name: jmeter-report
       path: results-html
   ```

3. Or in **Jenkinsfile**:

   ```groovy
   stage('JMeter Performance Test') {
       steps {
           sh 'jmeter -n -t kafka-performance-test.jmx -l results.jtl -e -o results-html'
           publishHTML([allowMissing: false,
                        keepAll: true,
                        reportDir: 'results-html',
                        reportFiles: 'index.html',
                        reportName: 'JMeter Report'])
       }
   }
   ```

---

# 🔹 5. Performance Metrics to Track

* **Producer throughput** (records/sec, MB/sec)
* **Producer latency** (avg, p95, p99)
* **Consumer throughput**
* **Consumer lag**
* **End-to-end latency** (event produced → processed → persisted)

---

✅ Now your CI/CD will:

* Run **unit → integration → e2e** tests
* Run **Kafka perf tests (built-in or JMeter)**
* Publish **reports (JUnit + JMeter)**

---

👉 Do you want me to also prepare a **Grafana + Prometheus dashboard setup** for **real-time Kafka performance monitoring** (outside CI/CD, in production)?


Nice — I’ll give you a compact, **practical end-to-end guide** to set up **Prometheus + Grafana monitoring for Kafka** (brokers, consumer groups, Streams apps) including a ready **docker-compose** example, `prometheus.yml`, JMX exporter notes, Grafana dashboard IDs to import, alerting rule examples, and troubleshooting tips. I’ll also point to a few high-quality references you can copy from.

> Sources you can reference while implementing: JMX Prometheus Exporter docs and examples, kafka\_exporter (and alternatives like kminion), community Docker examples, and Grafana dashboard templates. ([Zenika][1], [GitHub][2], [Robust Perception][3], [Grafana Labs][4])

---

# 1) Architecture & components (what you need)

* **JMX → Prometheus**: Use the **jmx\_prometheus\_javaagent** attached to each Kafka broker (and optionally ZooKeeper & Connect). This exposes broker JVM & Kafka metrics as `/metrics`. ([Zenika][1])
* **Kafka Exporter (consumer/offsets)**: A separate exporter (e.g. `danielqsj/kafka_exporter`, `deathowl/kafka_exporter`, or `kminion`) to surface consumer group offsets, lags, partition metrics that JMX alone doesn’t offer. ([GitHub][2])
* **Prometheus server**: Scrapes `/metrics` endpoints from JMX agents and exporters.
* **Grafana**: Visualize dashboards (ready dashboards available on Grafana.com). ([Grafana Labs][4])

---

# 2) Minimal docker-compose (quick start)

This is a compact composition used for local/dev monitoring (not production HA). Create `docker-compose.yml`:

```yaml
version: '3.7'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    ports: ["2181:2181"]

  kafka:
    image: confluentinc/cp-kafka:7.4.0
    depends_on: ["zookeeper"]
    environment:
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_BROKER_ID: 1
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    ports: ["9092:9092"]
    # in production add jmx agent to kafka start command or bake into image

  kafka-exporter:
    image: danielqsj/kafka_exporter:latest
    environment:
      KAFKA_SERVER: "kafka:9092"
    ports: ["9308:9308"]  # exporter /metrics at :9308/metrics
    depends_on: ["kafka"]

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    ports: ["9090:9090"]
    depends_on: ["kafka-exporter"]

  grafana:
    image: grafana/grafana:latest
    ports: ["3000:3000"]
    depends_on: ["prometheus"]
```

Notes: for real brokers, attach the `jmx_prometheus_javaagent` via JVM flags or a custom Docker image so each Kafka broker exposes JMX metrics. Example projects show this pattern. ([Robust Perception][3], [GitHub][5])

(For a more complete Docker example including JMX agent and Grafana dashboards, see the RobustPerception / example repos and community docker-compose projects.) ([Robust Perception][3], [GitHub][6])

---

# 3) `prometheus.yml` (scrape config)

Create `prometheus.yml` in the same folder:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'kafka-jmx'
    static_configs:
      - targets: ['kafka:7071']   # JMX exporter HTTP port per broker (adjust)
  
  - job_name: 'kafka-exporter'
    static_configs:
      - targets: ['kafka-exporter:9308']

  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```

* `kafka:7071` assumes you configured the JMX exporter on the broker to listen on port `7071`. Modify addresses/ports for each broker in multi-broker clusters. The JMX exporter runs *inside* the broker JVM and opens an HTTP endpoint. ([Zenika][1])

---

# 4) JMX exporter: quick config and attach

* Download the `jmx_prometheus_javaagent.jar` and a YAML config (lists mbeans to expose). Example agent args when starting Kafka broker JVM:

  ```
  -javaagent:/opt/jmx_exporter/jmx_prometheus_javaagent.jar=7071:/opt/jmx_exporter/kafka-jmx-config.yml
  ```
* A useful starter `kafka-jmx-config.yml` is available from community examples and includes Kafka server mbeans like `kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec` and consumer group metrics. See example projects for a complete config. ([Zenika][1], [Robust Perception][3])

Tip: start with a community config and prune to the metrics you need (too many metrics = heavy cardinality).

---

# 5) Kafka exporter (consumer/offsets)

* `danielqsj/kafka_exporter` (or `kminion`) queries the Kafka cluster and exposes:

  * topic offsets
  * consumer group offsets & lags
  * partition leadership info
* Run it pointing to your bootstrap server and then have Prometheus scrape its `/metrics` endpoint. ([GitHub][2])

Example run (container env already in docker-compose above): exporter exposes `http://kafka-exporter:9308/metrics`.

---

# 6) Grafana — dashboards to import

Use prebuilt dashboard templates (Grafana.com). Import by ID (Dashboard → Import → paste ID):

* **Kafka Exporter Overview** — ID: **7589**. Good for consumer & lag overview. ([GitHub][2], [Grafana Labs][7])
* **Kafka Exporter Overview (alternative)** — ID: **12931** (another variant). ([Grafana Labs][4])
* **Kminion / Kafka Cluster dashboards** — search Grafana for “Kafka” to find cluster/broker dashboards (IDs vary). ([Grafana Labs][8])

After importing, point panels’ Prometheus datasource to your Prometheus instance ([http://prometheus:9090](http://prometheus:9090) or the host URL).

---

# 7) Metrics and alerts to track (practical list)

**Broker / Streams metrics**

* `kafka_server_BrokerTopicMetrics_MessagesInPerSec` — incoming msg/sec.
* `kafka_server_BrokerTopicMetrics_BytesInPerSec` / BytesOut.
* JVM metrics: heap, GC pause, thread count. (exposed by JMX exporter). ([Zenika][1])

**Consumer / Lag metrics (from kafka\_exporter)**

* consumer group lag per topic partition (`kafka_consumergroup_lag` or exporter-specific metric)
* unconsumed offsets growth (rising lag) — trigger alert

**Important alerts (examples)**

* High consumer lag > threshold for > 5 min
* Broker down (scrape failures)
* Under replicated partitions > 0
* High GC pause > 1s for > 1 minute

Example alert rule (Prometheus `rules.yml`):

```yaml
groups:
- name: kafka-alerts
  rules:
  - alert: KafkaBrokerDown
    expr: up{job="kafka-jmx"} == 0
    for: 2m
    labels: { severity: "critical" }
    annotations:
      summary: "Kafka broker down"
      description: "Prometheus cannot scrape kafka-jmx endpoint for >2m"

  - alert: ConsumerLagHigh
    expr: sum(kafka_consumergroup_lag{group=~".+"}) > 1000
    for: 5m
    labels: { severity: "warning" }
    annotations:
      summary: "High consumer lag"
      description: "Total consumer group lag > 1000 for 5 minutes"
```

(Adjust metric names according to the exporter you use.) ([GitHub][2])

---

# 8) Troubleshooting checklist & tips

* **No metrics at `/metrics`**: verify JMX exporter agent is attached to Kafka JVM and listening on the configured port. Check broker start logs for `jmx_prometheus_javaagent` binding messages. ([Zenika][1])
* **High cardinality / too many metrics**: prune the `kafka-jmx-config.yml` to only match essential MBeans — avoid `*` wildcard that expands across topic-level MBeans unless needed. ([Robust Perception][3])
* **Consumer lag metrics missing**: kafka\_exporter must be able to connect to Kafka bootstrap servers and have ACL access (if enabled) to read consumer group offsets. Check exporter logs. ([GitHub][9])
* **Dashboards show no data**: ensure Grafana datasource is set to Prometheus URL and time range is correct (last 5m / 1h).
* **Prometheus scrape failing**: check `prometheus.yml` targets page (`http://<prometheus-host>:9090/targets`) for scrape errors.

---

# 9) Production considerations

* Run **one JMX exporter per broker JVM** (not one global). Expose it on an internal network only. ([Zenika][1])
* Use **scrape\_interval** appropriate for your SLA (15s or 30s).
* Add **relabelling** in Prometheus to avoid too many metric labels (reduce cardinality).
* Secure Prometheus/Grafana endpoints (auth, network rules).
* Consider managed solutions (Grafana Cloud / Confluent Cloud monitoring) if you want less ops. ([Grafana Labs][7])

---

# 10) Handy repo & reading links (copy/paste)

* kafka\_exporter (consumer/offset exporter): GitHub — danielqsj/kafka\_exporter. ([GitHub][2])
* JMX exporter (how to attach & config examples): Zenika workshop & robustperception examples. ([Zenika][1], [Robust Perception][3])
* Example docker-compose for Kafka + Prometheus + Grafana: community repos (search "kafka-prometheus-monitoring" on GitHub). ([GitHub][6])
* Grafana dashboards for Kafka Exporter: Grafana.com dashboard IDs (e.g., 7589, 12931). ([Grafana Labs][7])

---

If you want, I can:

* generate a **full, production-ready docker-compose** that includes a pre-baked Kafka image with the JMX agent and a ready Prometheus `rules.yml` and Grafana provisioning files, **or**
* create a **Grafana dashboard JSON** tuned for your vehicle-tracking topics/metrics (e.g., topic throughput, consumer lag per service) that you can import directly.

Which one should I do next?

[1]: https://zenika.github.io/workshop-monitor-kafka/5_JMX_EXPORTER_PROMETHEUS.html?utm_source=chatgpt.com "JMX Exporter & Prometheus | Workshop Monitoring Kafka"
[2]: https://github.com/danielqsj/kafka_exporter?utm_source=chatgpt.com "danielqsj/kafka_exporter: Kafka exporter for Prometheus"
[3]: https://www.robustperception.io/monitoring-kafka-in-a-docker-container-using-prometheus?utm_source=chatgpt.com "Monitoring Kafka in a Docker container using Prometheus"
[4]: https://grafana.com/grafana/dashboards/12931-kafka-exporter-overview/?utm_source=chatgpt.com "Kafka Exporter Overview | Grafana Labs"
[5]: https://github.com/rama-nallamilli/kafka-prometheus-monitoring?utm_source=chatgpt.com "rama-nallamilli/kafka-prometheus-monitoring"
[6]: https://github.com/jeanlouisboudart/kafka-platform-prometheus/blob/master/docker-compose.yml?utm_source=chatgpt.com "kafka-platform-prometheus/docker-compose.yml at master"
[7]: https://grafana.com/grafana/dashboards/18941-kafka-exporter-overview/?utm_source=chatgpt.com "Kafka Exporter Dashboard | Grafana Labs"
[8]: https://grafana.com/grafana/dashboards/14014-kminion-groups/?utm_source=chatgpt.com "Prometheus Exporter for Apache Kafka"
[9]: https://github.com/orgs/strimzi/discussions/7553?utm_source=chatgpt.com "How to Configure Promethus to Pull/Scrape Kafkaexporter ..."
