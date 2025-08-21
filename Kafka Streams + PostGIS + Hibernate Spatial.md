Got it ✅ You want **stepwise notes + code** for using **Kafka Streams + PostGIS + Hibernate Spatial in a Spring Boot project**.
Let’s break it down like lecture notes so you can study + implement easily.

---

# 📘 Notes: Kafka Streams + PostGIS + Hibernate Spatial in Spring Boot

---

## 1. **What is PostGIS?**

* **PostGIS** is a spatial database extender for PostgreSQL.
* It allows storing & querying **geospatial data** (points, lines, polygons).
* Example: Storing locations of vehicles, routes, or areas.
* Functions: `ST_Distance`, `ST_Within`, `ST_Intersects` → used to check distance, inside area, intersections.

---

## 2. **What is Hibernate Spatial?**

* Hibernate ORM extension for handling **geospatial data types**.
* Provides Java bindings to PostGIS functions.
* Works with `Geometry` objects (Point, LineString, Polygon).
* Uses **JTS (Java Topology Suite)** for geometry modeling.

---

## 3. **What is Kafka Streams?**

* **Kafka Streams** = library for processing real-time data streams.
* Reads from Kafka topic → applies transformations → writes to another topic or DB.
* In GIS use cases:

  * Consume **location updates** (vehicle GPS).
  * Process in real time (detect area, nearest neighbors).
  * Store processed location into PostGIS.

---

## 4. **Typical Use Case**

👉 "Real-Time Vehicle Tracking"

* IoT devices send GPS → Kafka topic `vehicle-location`.
* Kafka Streams consumes and enriches location data.
* Store in **PostGIS** using Hibernate Spatial.
* Query: "Which vehicles are within 5km of city center?"

---

## 5. **Spring Boot Project Setup**

### Dependencies (`pom.xml`)

```xml
<dependencies>
    <!-- Spring Boot -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>

    <!-- Kafka Streams -->
    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka</artifactId>
    </dependency>

    <!-- Hibernate + JPA -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <!-- PostgreSQL -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
    </dependency>

    <!-- Hibernate Spatial -->
    <dependency>
        <groupId>org.hibernate</groupId>
        <artifactId>hibernate-spatial</artifactId>
    </dependency>

    <!-- JTS Geometry -->
    <dependency>
        <groupId>org.locationtech.jts</groupId>
        <artifactId>jts-core</artifactId>
    </dependency>
</dependencies>
```

---

## 6. **Application Properties**

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/geodb
spring.datasource.username=postgres
spring.datasource.password=postgres
spring.jpa.hibernate.ddl-auto=update
spring.jpa.database-platform=org.hibernate.spatial.dialect.postgis.PostgisDialect

# Kafka
spring.kafka.bootstrap-servers=localhost:9092
```

---

## 7. **Entity with Hibernate Spatial**

```java
import jakarta.persistence.*;
import org.locationtech.jts.geom.Point;

@Entity
@Table(name = "vehicles")
public class Vehicle {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String vehicleId;

    // Spatial column
    @Column(columnDefinition = "Geometry(Point, 4326)")
    private Point location;

    // Getters & setters
}
```

---

## 8. **Repository for Spatial Queries**

```java
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.locationtech.jts.geom.Point;

import java.util.List;

public interface VehicleRepository extends JpaRepository<Vehicle, Long> {

    // Find vehicles within X meters of a given point
    @Query(value = "SELECT * FROM vehicles v WHERE ST_DWithin(v.location, :point, :distance)", nativeQuery = true)
    List<Vehicle> findNearbyVehicles(Point point, double distance);
}
```

---

## 9. **Kafka Streams Configuration**

```java
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.kstream.KStream;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class KafkaStreamsConfig {

    @Bean
    public KStream<String, String> kStream(StreamsBuilder builder) {
        KStream<String, String> stream = builder.stream("vehicle-location");

        stream.foreach((key, value) -> {
            System.out.println("Received location: " + value);
            // Here you parse JSON -> Vehicle -> Save to DB
        });

        return stream;
    }
}
```

---

## 10. **Service Layer: Save Kafka Data to PostGIS**

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import org.locationtech.jts.geom.GeometryFactory;
import org.locationtech.jts.geom.Point;
import org.locationtech.jts.geom.Coordinate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class VehicleService {

    private final VehicleRepository repo;
    private final ObjectMapper objectMapper = new ObjectMapper();
    private final GeometryFactory geometryFactory = new GeometryFactory();

    public VehicleService(VehicleRepository repo) {
        this.repo = repo;
    }

    @Transactional
    public void saveLocation(String json) {
        try {
            VehicleDto dto = objectMapper.readValue(json, VehicleDto.class);
            Point point = geometryFactory.createPoint(new Coordinate(dto.getLongitude(), dto.getLatitude()));
            point.setSRID(4326);

            Vehicle vehicle = new Vehicle();
            vehicle.setVehicleId(dto.getVehicleId());
            vehicle.setLocation(point);

            repo.save(vehicle);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

---

## 11. **DTO for Kafka Messages**

```java
public class VehicleDto {
    private String vehicleId;
    private double latitude;
    private double longitude;

    // getters & setters
}
```

---

## 12. **Integration: Kafka → DB**

Update KafkaStreamsConfig:

```java
@Bean
public KStream<String, String> kStream(StreamsBuilder builder, VehicleService service) {
    KStream<String, String> stream = builder.stream("vehicle-location");

    stream.foreach((key, value) -> {
        service.saveLocation(value);
    });

    return stream;
}
```

---

## 13. **Test with Producer**

Send a test message:

```bash
kafka-console-producer --broker-list localhost:9092 --topic vehicle-location
>{"vehicleId":"V1","latitude":28.7041,"longitude":77.1025}
```

---

## 14. **Query Example**

```java
@RestController
@RequestMapping("/vehicles")
public class VehicleController {

    private final VehicleRepository repo;
    private final GeometryFactory geometryFactory = new GeometryFactory();

    public VehicleController(VehicleRepository repo) {
        this.repo = repo;
    }

    @GetMapping("/nearby")
    public List<Vehicle> getNearby(
            @RequestParam double lat,
            @RequestParam double lon,
            @RequestParam double distance) {

        Point point = geometryFactory.createPoint(new Coordinate(lon, lat));
        point.setSRID(4326);

        return repo.findNearbyVehicles(point, distance);
    }
}
```

---

# ✅ End-to-End Flow

1. IoT Device → sends GPS to Kafka (`vehicle-location` topic).
2. Kafka Streams → consumes JSON, maps to `Vehicle`.
3. Hibernate Spatial + PostGIS → stores geolocation as `Point`.
4. REST API → queries vehicles within distance.

---

👉 This gives you **Kafka Streams + PostGIS + Hibernate Spatial integration in Spring Boot** for real-time geospatial apps.



---
Perfect 👍 Let’s extend the notes with a **Docker Compose setup** for running Kafka + Zookeeper + PostgreSQL + PostGIS locally so you can fully test this **Spring Boot + Kafka Streams + PostGIS** project on your PC.

---

# ⚡ Docker Compose Setup for Kafka + PostGIS

---

## 1. **docker-compose.yml**

```yaml
version: '3.8'
services:

  # Zookeeper (required by Kafka)
  zookeeper:
    image: confluentinc/cp-zookeeper:7.3.0
    container_name: zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"

  # Kafka Broker
  kafka:
    image: confluentinc/cp-kafka:7.3.0
    container_name: kafka
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

  # PostgreSQL with PostGIS
  postgres:
    image: postgis/postgis:15-3.3
    container_name: postgres
    environment:
      POSTGRES_DB: geodb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

---

## 2. **Start Services**

Run:

```bash
docker-compose up -d
```

This will start:

* Zookeeper → `localhost:2181`
* Kafka Broker → `localhost:9092`
* Postgres + PostGIS → `localhost:5432`

---

## 3. **Check Containers**

```bash
docker ps
```

You should see `zookeeper`, `kafka`, and `postgres`.

---

## 4. **Create Kafka Topic**

```bash
docker exec -it kafka kafka-topics \
  --create --topic vehicle-location \
  --bootstrap-server localhost:9092 \
  --partitions 1 \
  --replication-factor 1
```

Check topics:

```bash
docker exec -it kafka kafka-topics --list --bootstrap-server localhost:9092
```

---

## 5. **Connect to Postgres**

```bash
docker exec -it postgres psql -U postgres -d geodb
```

Enable PostGIS inside the database:

```sql
CREATE EXTENSION postgis;
```

---

## 6. **Test Kafka Producer**

```bash
docker exec -it kafka kafka-console-producer \
  --broker-list localhost:9092 \
  --topic vehicle-location
```

Send message:

```json
{"vehicleId":"V1","latitude":28.7041,"longitude":77.1025}
```

---

## 7. **Check Spring Boot**

* Boot app should consume the Kafka event.
* Store location as `Point` in PostGIS.
* Query via REST:

```
GET http://localhost:8080/vehicles/nearby?lat=28.7041&lon=77.1025&distance=5000
```

---

# ✅ End-to-End Working Flow

1. **Kafka + Zookeeper** → handle messages.
2. **Postgres + PostGIS** → store spatial data.
3. **Spring Boot (Kafka Streams + Hibernate Spatial)** → consume → process → persist.
4. **REST API** → query spatial data (`findNearbyVehicles`).

---

👉 With this `docker-compose.yml`, you have a complete local development environment for your **real-time geospatial project** 🚀

Would you like me to also prepare a **stepwise testing plan** (send multiple locations, query radius, see live updates) so you can verify each integration works properly?

---


---
Great 🚀 Let’s make a **Stepwise Testing Plan** so you can verify that your whole **Spring Boot + Kafka Streams + PostGIS** project is working correctly with Docker Compose.

---

# 🧪 Stepwise Testing Plan

---

## **Step 1: Start Infra**

```bash
docker-compose up -d
```

* Runs Zookeeper, Kafka, Postgres+PostGIS.
* Verify with:

```bash
docker ps
```

You should see 3 containers: `zookeeper`, `kafka`, `postgres`.

---

## **Step 2: Enable PostGIS**

Enter Postgres:

```bash
docker exec -it postgres psql -U postgres -d geodb
```

Run:

```sql
CREATE EXTENSION IF NOT EXISTS postgis;
```

✅ Confirms PostGIS enabled.

---

## **Step 3: Create Kafka Topic**

```bash
docker exec -it kafka kafka-topics \
  --create --topic vehicle-location \
  --bootstrap-server localhost:9092 \
  --partitions 1 \
  --replication-factor 1
```

Check topic exists:

```bash
docker exec -it kafka kafka-topics --list --bootstrap-server localhost:9092
```

✅ `vehicle-location` should appear.

---

## **Step 4: Start Spring Boot App**

Run your Boot app:

```bash
mvn spring-boot:run
```

or

```bash
./gradlew bootRun
```

Console should show:

```
Started KafkaStreamsConfig
Listening on topic: vehicle-location
```

---

## **Step 5: Send Location Events**

Open Kafka producer:

```bash
docker exec -it kafka kafka-console-producer \
  --broker-list localhost:9092 \
  --topic vehicle-location
```

Send some JSON:

```json
{"vehicleId":"V1","latitude":28.7041,"longitude":77.1025}
{"vehicleId":"V2","latitude":28.5355,"longitude":77.3910}
{"vehicleId":"V3","latitude":28.4595,"longitude":77.0266}
```

✅ Spring Boot logs should show:

```
Received location: {"vehicleId":"V1",...}
Saved vehicle V1
```

---

## **Step 6: Verify Data in PostGIS**

Connect to Postgres:

```bash
docker exec -it postgres psql -U postgres -d geodb
```

Run:

```sql
SELECT vehicle_id, ST_AsText(location) FROM vehicles;
```

Example output:

```
 V1 | POINT(77.1025 28.7041)
 V2 | POINT(77.3910 28.5355)
 V3 | POINT(77.0266 28.4595)
```

✅ Confirms spatial data stored.

---

## **Step 7: Query Nearby Vehicles**

Use REST API:

```
GET http://localhost:8080/vehicles/nearby?lat=28.7041&lon=77.1025&distance=50000
```

Expected JSON response:

```json
[
  {"id":1,"vehicleId":"V1","location":"POINT(77.1025 28.7041)"},
  {"id":2,"vehicleId":"V2","location":"POINT(77.3910 28.5355)"}
]
```

✅ Returns vehicles within 50km of Delhi.

---

## **Step 8: Add Another Event (Live Update)**

Send another GPS update for V1:

```json
{"vehicleId":"V1","latitude":28.4500,"longitude":77.1200}
```

Check DB:

```sql
SELECT vehicle_id, ST_AsText(location) FROM vehicles WHERE vehicle_id='V1';
```

✅ Vehicle V1 location should be updated.

---

## **Step 9: Edge Testing**

1. Send invalid JSON → should be logged as error (not crash).
2. Send location with extreme coords (lat=91) → should fail gracefully.
3. Query REST with small radius (100m) → only nearby vehicles returned.

---

## **Step 10: Real-Time Flow Confirmed**

* Kafka → receives messages.
* Spring Boot (Kafka Streams) → consumes + saves.
* Hibernate Spatial + PostGIS → stores geospatial data.
* REST API → performs `ST_DWithin` queries.

---

✅ With this stepwise test plan, you can **prove each integration works** before moving to production.

---

Would you like me to also prepare a **Kibana/Grafana monitoring dashboard setup** (visualizing vehicle movements on a map) as the **next step** after this?

---


---

Awesome 🔥 Now let’s extend your project with a **Monitoring Dashboard** so you can **visualize real-time vehicle movements on a map** using **Grafana + PostGIS**.

---

# 🌍 Real-Time Vehicle Tracking Dashboard (Grafana + PostGIS)

---

## 1. **Why Grafana with PostGIS?**

* Grafana can connect directly to **PostgreSQL/PostGIS**.
* You can build **real-time dashboards** with maps.
* Perfect for IoT / GPS tracking → vehicles, assets, delivery systems.

---

## 2. **Extend docker-compose.yml**

Add Grafana + Adminer (for easy DB inspection):

```yaml
  grafana:
    image: grafana/grafana:10.0.3
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
    depends_on:
      - postgres

  adminer:
    image: adminer
    container_name: adminer
    ports:
      - "8080:8080"
    depends_on:
      - postgres
```

Restart services:

```bash
docker-compose up -d
```

✅ Now you have:

* Grafana → `http://localhost:3000` (admin / admin)
* Adminer → `http://localhost:8080` (DB GUI, login with postgres/postgres)

---

## 3. **Configure Grafana Data Source**

1. Go to `http://localhost:3000`
2. Login → `admin/admin`
3. Add **Postgres Data Source**:

   * Host: `postgres:5432`
   * Database: `geodb`
   * User: `postgres`
   * Password: `postgres`
   * SSL: disable

✅ Grafana now connected to PostGIS.

---

## 4. **Install Worldmap Panel Plugin**

Grafana supports **map visualizations**:

```bash
docker exec -it grafana grafana-cli plugins install grafana-worldmap-panel
docker restart grafana
```

---

## 5. **Create Dashboard**

1. Go to **Dashboards → New → Panel**
2. Select **Postgres** as source
3. Use query to get live vehicle data:

```sql
SELECT
  vehicle_id AS "label",
  ST_Y(location::geometry) AS latitude,
  ST_X(location::geometry) AS longitude,
  NOW() AS "time"
FROM vehicles;
```

4. Visualization → **Worldmap Panel**

   * Location mode: **latitude/longitude**
   * Value: vehicle\_id
   * Show labels: yes

✅ You’ll see vehicles as **dots on the map**, updated in real time.

---

## 6. **Enable Auto Refresh**

At the top of Grafana dashboard:

* Set **Refresh Interval**: `5s` (or `10s`)
* Vehicles move in real-time as Kafka produces new messages.

---

## 7. **End-to-End Flow**

1. IoT → Kafka → Spring Boot → PostGIS.
2. Grafana queries PostGIS → maps `latitude/longitude`.
3. Worldmap updates every few seconds.
4. You see vehicles moving on live dashboard.

---

## 8. **Optional Enhancements**

* **GeoJSON export**: REST API can serve GeoJSON so frontends (Leaflet, Mapbox, OpenLayers) can render.
* **Alerts**: Grafana alert when vehicle leaves a geofence (`ST_Within`).
* **Time-series replay**: Store historical positions and animate routes.

---

✅ With this, you now have:

* **Kafka Streams** → Real-time processing
* **PostGIS** → Spatial database
* **Hibernate Spatial** → ORM integration
* **Grafana** → Live map dashboard

---

👉 Do you want me to also prepare a **Leaflet.js web UI** (instead of Grafana) where you can show vehicles on an interactive map with your own Spring Boot API? That would give you a **custom web dashboard** instead of Grafana.

---




---
Perfect 👍 Let’s now extend the project with a **custom Web Dashboard using Leaflet.js** (instead of only Grafana).
This will allow you to **visualize vehicles on an interactive map** using data served by your **Spring Boot REST API**.

---

# 🌍 Leaflet.js + Spring Boot (Custom Vehicle Tracking Dashboard)

---

## 1. **Why Leaflet.js?**

* Lightweight open-source **map visualization library**.
* Works with **OpenStreetMap** (free) → no Google Maps license needed.
* Very easy to embed in a Spring Boot project (as static HTML/JS).

---

## 2. **Spring Boot: REST API for GeoJSON**

Since Leaflet can consume **GeoJSON**, let’s make an API that returns vehicles as GeoJSON.

### Controller

```java
import org.springframework.web.bind.annotation.*;
import java.util.*;

@RestController
@RequestMapping("/map")
public class MapController {

    private final VehicleRepository repo;

    public MapController(VehicleRepository repo) {
        this.repo = repo;
    }

    @GetMapping("/vehicles")
    public Map<String, Object> getVehicles() {
        List<Vehicle> vehicles = repo.findAll();

        List<Map<String, Object>> features = new ArrayList<>();
        for (Vehicle v : vehicles) {
            Map<String, Object> feature = new HashMap<>();
            feature.put("type", "Feature");

            Map<String, Object> geometry = new HashMap<>();
            geometry.put("type", "Point");
            geometry.put("coordinates", new double[]{
                    v.getLocation().getX(),  // longitude
                    v.getLocation().getY()   // latitude
            });
            feature.put("geometry", geometry);

            Map<String, Object> properties = new HashMap<>();
            properties.put("vehicleId", v.getVehicleId());
            feature.put("properties", properties);

            features.add(feature);
        }

        Map<String, Object> geojson = new HashMap<>();
        geojson.put("type", "FeatureCollection");
        geojson.put("features", features);

        return geojson;
    }
}
```

✅ This exposes endpoint:

```
GET http://localhost:8080/map/vehicles
```

Response (GeoJSON):

```json
{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "geometry": {"type": "Point", "coordinates": [77.1025, 28.7041]},
      "properties": {"vehicleId": "V1"}
    }
  ]
}
```

---

## 3. **Frontend (Leaflet.js)**

Create a file `src/main/resources/static/index.html`:

```html
<!DOCTYPE html>
<html>
<head>
  <title>Vehicle Tracking Dashboard</title>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0">

  <!-- Leaflet CSS -->
  <link rel="stylesheet" href="https://unpkg.com/leaflet/dist/leaflet.css" />
  <style>
    #map { height: 100vh; }
  </style>
</head>
<body>
  <div id="map"></div>

  <!-- Leaflet JS -->
  <script src="https://unpkg.com/leaflet/dist/leaflet.js"></script>
  <script>
    // Initialize map
    var map = L.map('map').setView([28.7041, 77.1025], 10); // Delhi center

    // Add OpenStreetMap tiles
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
      maxZoom: 19,
    }).addTo(map);

    // Function to load vehicles from backend
    async function loadVehicles() {
      const response = await fetch('/map/vehicles');
      const data = await response.json();

      // Clear old markers
      if (window.vehicleLayer) {
        map.removeLayer(window.vehicleLayer);
      }

      // Add GeoJSON layer
      window.vehicleLayer = L.geoJSON(data, {
        onEachFeature: function (feature, layer) {
          layer.bindPopup("Vehicle ID: " + feature.properties.vehicleId);
        }
      }).addTo(map);
    }

    // Load vehicles every 5 seconds
    loadVehicles();
    setInterval(loadVehicles, 5000);
  </script>
</body>
</html>
```

✅ Now open:
👉 `http://localhost:8080/index.html`
You’ll see an **interactive map** with vehicle locations as markers.

---

## 4. **How It Works**

1. **Spring Boot REST** serves vehicle data as GeoJSON.
2. **Leaflet.js frontend** fetches `/map/vehicles` every 5 seconds.
3. Map auto-updates with **new vehicle positions** from PostGIS.

---

## 5. **Optional Enhancements**

* Show **vehicle trails (LineString)** by storing historical positions.
* Use **different marker colors** for vehicle types.
* Implement **geofencing**: highlight if vehicle enters restricted area (`ST_Within`).
* Replace OpenStreetMap tiles with **Mapbox/Google Maps** if needed.

---

✅ Now you have:

* **Kafka Streams + PostGIS** → Real-time data pipeline.
* **Grafana Dashboard** → quick monitoring & alerts.
* **Leaflet.js Web UI** → custom, interactive map dashboard.

---

Would you like me to also prepare a **mobile-friendly version (Ionic/React Native)** of this Leaflet dashboard so you can track vehicles from your phone 📱?


---
Do you want me to also prepare **Docker Compose setup** (Kafka + PostgreSQL + PostGIS) so you can run this project fully on your PC?
