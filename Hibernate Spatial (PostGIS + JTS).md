# Hibernate Spatial — step-wise notes + Spring Boot example (PostGIS + JTS)

Nice — below is a compact, practical step-by-step guide you can follow to add **Hibernate Spatial** to a Spring Boot app (PostgreSQL + PostGIS + JTS). I’ll explain what it is, point out pitfalls, and give ready-to-copy code (entity, repo, service, controller, SQL). Key references are attached after each important section.

---

## TL;DR (what Hibernate Spatial is)

* **Hibernate Spatial** is an extension to Hibernate ORM that adds first-class support for geospatial types, functions and dialects so you can persist/query `Point`, `LineString`, `Polygon`, etc., using Java geometry objects. ([hibernatespatial.org][1])
* It commonly works together with **PostGIS** (PostgreSQL spatial extension) and **JTS** (Java Topology Suite) for geometry modelling and operations. ([GitHub][2], [PostGIS][3])

---

## 1) Prerequisites (what you need)

1. PostgreSQL server with **PostGIS** installed/enabled in the database you will use. (`CREATE EXTENSION postgis;`) ([PostGIS][3])
2. A Spring Boot project (Maven/Gradle) with Spring Data JPA.
3. Add **hibernate-spatial** and **JTS** to your dependencies (example below). ([Maven Central][4])

---

## 2) Maven dependencies (example)

Let Spring Boot manage versions when possible. If you must pin, make the `hibernate-spatial` version match your Hibernate core version.

```xml
<!-- pom.xml (excerpt) -->
<dependencies>
  <!-- Spring Data JPA -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
  </dependency>

  <!-- PostgreSQL JDBC -->
  <dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
  </dependency>

  <!-- Hibernate Spatial (artifact relocated to org.hibernate.orm) -->
  <dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-spatial</artifactId>
    <!-- omit version to inherit from Boot BOM, or set matching version -->
  </dependency>

  <!-- JTS Topology Suite -->
  <dependency>
    <groupId>org.locationtech.jts</groupId>
    <artifactId>jts</artifactId>
  </dependency>
</dependencies>
```

(References: Hibernate Spatial on Maven/Central; JTS.) ([Maven Central][4])

---

## 3) DB: enable PostGIS and create table (SQL)

Make sure PostGIS is enabled **before** Hibernate tries to auto-create tables (otherwise you may get `type "geometry" does not exist`). Example:

```sql
-- run as superuser in your DB
CREATE EXTENSION IF NOT EXISTS postgis;

CREATE TABLE locations (
  id SERIAL PRIMARY KEY,
  name TEXT,
  geom geometry(Point,4326)   -- SRID 4326 (WGS84)
);

CREATE INDEX idx_locations_geom ON locations USING GIST (geom);
```

* Use `USING GIST` to create a spatial index for fast spatial queries. ([PostGIS][3])

**Tip:** If you use `spring.jpa.hibernate.ddl-auto=create`/`update`, enable PostGIS first (or create the table manually) to avoid schema creation errors. (Common pitfall.) ([Stack Overflow][5])

---

## 4) application.properties example (Spring Boot)

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/geodb
spring.datasource.username=postgres
spring.datasource.password=postgres

# Use the PostGIS hibernate dialect so spatial functions and types are supported
spring.jpa.properties.hibernate.dialect=org.hibernate.spatial.dialect.postgis.PostgisDialect

# dev convenience (use carefully)
spring.jpa.hibernate.ddl-auto=update
```

The `PostgisDialect` extension adds spatial function/operator mapping into Hibernate. ([JBoss Documentation][6])

---

## 5) Entity mapping (JTS `Point`)

You can map a JTS geometry type directly on a field. Many modern versions of Hibernate Spatial **do not require** a custom `@Type`—just declare the JTS type and set `columnDefinition` to the geometry SQL type.

```java
package com.example.geo;

import jakarta.persistence.*;
import org.locationtech.jts.geom.Point;

@Entity
@Table(name = "locations")
public class Location {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    // store as geometry(Point, 4326)
    @Column(columnDefinition = "geometry(Point,4326)")
    private Point geom;

    // getters / setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }

    public Point getGeom() { return geom; }
    public void setGeom(Point geom) { this.geom = geom; }
}
```

Mapping notes: declare SRID in `columnDefinition` (4326 = WGS84). Hibernate Spatial will map the `Point` property to the PostGIS `geometry` column. ([JBoss Documentation][7], [Gist][8])

---

## 6) Repository + spatial query example

Example: find locations within N meters of a lon/lat. For accurate meter distances cast to `geography` or project coordinates appropriately.

```java
package com.example.geo;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import java.util.List;

public interface LocationRepository extends JpaRepository<Location, Long> {

    // nativeQuery example using ST_DWithin and geography cast for meters
    @Query(value = "SELECT * FROM locations l " +
                   "WHERE ST_DWithin(l.geom::geography, ST_SetSRID(ST_MakePoint(:lon, :lat), 4326)::geography, :meters)",
           nativeQuery = true)
    List<Location> findNearby(double lon, double lat, double meters);
}
```

* Prefer `ST_DWithin` for radius queries — it is index-aware and efficient. ([PostGIS][9])

---

## 7) Service: creating and saving a Point (JTS)

Use `GeometryFactory` (JTS) to create geometry objects to persist.

```java
package com.example.geo;

import org.locationtech.jts.geom.*;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class LocationService {
    private final LocationRepository repo;
    private final GeometryFactory gf = new GeometryFactory(new PrecisionModel(), 4326);

    public LocationService(LocationRepository repo) {
        this.repo = repo;
    }

    @Transactional
    public Location create(String name, double lon, double lat) {
        Point p = gf.createPoint(new Coordinate(lon, lat)); // lon = x, lat = y
        Location loc = new Location();
        loc.setName(name);
        loc.setGeom(p);
        return repo.save(loc);
    }
}
```

Note: JTS `Coordinate(x, y)` uses `(x=lon, y=lat)` for typical lon/lat (WGS84) usage.

---

## 8) Controller (simple REST)

```java
package com.example.geo;

import org.springframework.web.bind.annotation.*;
import java.util.List;

@RestController
@RequestMapping("/api/locations")
public class LocationController {
    private final LocationService svc;
    private final LocationRepository repo;

    public LocationController(LocationService svc, LocationRepository repo) {
        this.svc = svc;
        this.repo = repo;
    }

    @PostMapping
    public Location create(@RequestParam String name,
                           @RequestParam double lon,
                           @RequestParam double lat) {
        return svc.create(name, lon, lat);
    }

    @GetMapping("/nearby")
    public List<Location> nearby(@RequestParam double lon,
                                 @RequestParam double lat,
                                 @RequestParam(defaultValue = "1000") double meters) {
        return repo.findNearby(lon, lat, meters);
    }
}
```

---

## 9) Common pitfalls & tips

* **`geometry type does not exist` on startup**: usually caused by PostGIS extension not enabled in the DB before Hibernate runs schema creation—run `CREATE EXTENSION postgis;` first. ([PostGIS][3], [Stack Overflow][5])
* **Dialect mismatch**: set `spring.jpa.properties.hibernate.dialect` to the PostGIS dialect to ensure function mapping. ([JBoss Documentation][6])
* **Version compatibility**: match `hibernate-spatial` to your Hibernate core version; if you use Spring Boot, prefer the Boot-managed BOM or check Boot’s dependency versions. If you override dependencies, you can create runtime errors. (See Spring Boot dependency management notes.) ([Maven Central][4], [Maven Repository][10])
* **Indexing**: create a GiST index (`USING GIST`) on geometry columns—PostGIS functions like `ST_DWithin` will use the index as a pre-filter. ([PostGIS][11])

---

## 10) Quick test checklist

1. Confirm `psql -d geodb -c "SELECT PostGIS_full_version();"` returns a version.
2. Create extension if missing: `CREATE EXTENSION postgis;`. ([PostGIS][3])
3. Start Spring Boot app — check tables created and `geom` column type is `geometry`.
4. Insert an item (POST `/api/locations`) and query nearby endpoint.

---

## 11) Useful links & further reading

* Hibernate Spatial documentation (dialects & usage). ([hibernatespatial.org][1])
* `hibernate-spatial` on Maven/Central (artifact / versions). ([Maven Central][4])
* JTS Topology Suite (library & releases). ([GitHub][2], [Maven Central][12])
* PostGIS docs: enabling extension and spatial functions (e.g. `ST_DWithin`, `ST_DistanceSphere`). ([PostGIS][3])
* Intro article: Baeldung — *Introduction to Hibernate Spatial* (good conceptual overview). ([Baeldung on Kotlin][13])

---

If you want, I can now:

* generate a **full sample Spring Boot project** (pom, application.properties, complete Java classes) you can drop into your IDE, **or**
* adapt the code to a specific **Spring Boot version** you use (tell me the Boot version and whether you want Gradle or Maven) so I pin compatible versions.

Which would you like next?

[1]: https://www.hibernatespatial.org/documentation/documentation/?utm_source=chatgpt.com "Documentation"
[2]: https://github.com/locationtech/jts?utm_source=chatgpt.com "locationtech/jts: The JTS Topology Suite is a Java library ..."
[3]: https://postgis.net/documentation/getting_started/?utm_source=chatgpt.com "Getting Started"
[4]: https://central.sonatype.com/artifact/org.hibernate.orm/hibernate-spatial?utm_source=chatgpt.com "org.hibernate.orm:hibernate-spatial - Maven Central"
[5]: https://stackoverflow.com/questions/71009549/how-to-make-spring-boot-spatial-hibernate-and-postgis-work?utm_source=chatgpt.com "How to make spring boot, spatial hibernate and postgis ..."
[6]: https://docs.jboss.org/hibernate/orm/5.5/javadocs/org/hibernate/spatial/dialect/postgis/package-summary.html?utm_source=chatgpt.com "Package org.hibernate.spatial.dialect.postgis Description"
[7]: https://docs.jboss.org/hibernate/orm/5.2/userguide/html_single/chapters/query/spatial/Spatial.html?utm_source=chatgpt.com "Spatial"
[8]: https://gist.github.com/michael-simons/824ff17aebc1aa6fe4cf26a3fe795892?utm_source=chatgpt.com "An example on how to use Hibernate-Spatial with Spring ..."
[9]: https://postgis.net/docs/ST_DWithin.html?utm_source=chatgpt.com "ST_DWithin - PostGIS"
[10]: https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-dependencies/2.7.18?utm_source=chatgpt.com "2.7.18 - spring-boot-dependencies"
[11]: https://postgis.net/documentation/faq/spatial-indexes/?utm_source=chatgpt.com "How do I use spatial indexes?"



---
Got it 👍 You want me to prepare a **full runnable Spring Boot sample project** for Hibernate Spatial (PostGIS + JTS).
I’ll give you everything stepwise: `pom.xml`, `application.properties`, and Java classes.

---

# ✅ Spring Boot + Hibernate Spatial (PostGIS) Example Project

### 1) Project structure

```
geo-demo/
 ├── src/main/java/com/example/geo/
 │    ├── GeoDemoApplication.java
 │    ├── entity/Location.java
 │    ├── repo/LocationRepository.java
 │    ├── service/LocationService.java
 │    └── controller/LocationController.java
 ├── src/main/resources/application.properties
 └── pom.xml
```

---

### 2) `pom.xml`

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>geo-demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>
    <name>geo-demo</name>
    <description>Spring Boot + Hibernate Spatial + PostGIS demo</description>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.3.2</version> <!-- adjust if needed -->
        <relativePath/> 
    </parent>

    <properties>
        <java.version>17</java.version>
    </properties>

    <dependencies>
        <!-- Spring Data JPA -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>

        <!-- PostgreSQL driver -->
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
        </dependency>

        <!-- Hibernate Spatial -->
        <dependency>
            <groupId>org.hibernate.orm</groupId>
            <artifactId>hibernate-spatial</artifactId>
        </dependency>

        <!-- JTS Topology Suite -->
        <dependency>
            <groupId>org.locationtech.jts</groupId>
            <artifactId>jts</artifactId>
        </dependency>

        <!-- Spring Web (for REST controller) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
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

### 3) `application.properties`

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/geodb
spring.datasource.username=postgres
spring.datasource.password=postgres

# PostGIS dialect
spring.jpa.properties.hibernate.dialect=org.hibernate.spatial.dialect.postgis.PostgisDialect

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
```

---

### 4) Main class

```java
package com.example.geo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class GeoDemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(GeoDemoApplication.class, args);
    }
}
```

---

### 5) Entity

```java
package com.example.geo.entity;

import jakarta.persistence.*;
import org.locationtech.jts.geom.Point;

@Entity
@Table(name = "locations")
public class Location {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    // PostGIS geometry type (Point with SRID 4326)
    @Column(columnDefinition = "geometry(Point,4326)")
    private Point geom;

    // getters & setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }

    public Point getGeom() { return geom; }
    public void setGeom(Point geom) { this.geom = geom; }
}
```

---

### 6) Repository

```java
package com.example.geo.repo;

import com.example.geo.entity.Location;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;

import java.util.List;

public interface LocationRepository extends JpaRepository<Location, Long> {

    @Query(value = "SELECT * FROM locations l " +
                   "WHERE ST_DWithin(l.geom::geography, " +
                   "ST_SetSRID(ST_MakePoint(:lon, :lat),4326)::geography, :meters)",
           nativeQuery = true)
    List<Location> findNearby(double lon, double lat, double meters);
}
```

---

### 7) Service

```java
package com.example.geo.service;

import com.example.geo.entity.Location;
import com.example.geo.repo.LocationRepository;
import org.locationtech.jts.geom.*;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class LocationService {
    private final LocationRepository repo;
    private final GeometryFactory gf = new GeometryFactory(new PrecisionModel(), 4326);

    public LocationService(LocationRepository repo) {
        this.repo = repo;
    }

    @Transactional
    public Location create(String name, double lon, double lat) {
        Point p = gf.createPoint(new Coordinate(lon, lat));
        Location loc = new Location();
        loc.setName(name);
        loc.setGeom(p);
        return repo.save(loc);
    }
}
```

---

### 8) Controller

```java
package com.example.geo.controller;

import com.example.geo.entity.Location;
import com.example.geo.repo.LocationRepository;
import com.example.geo.service.LocationService;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/locations")
public class LocationController {
    private final LocationService svc;
    private final LocationRepository repo;

    public LocationController(LocationService svc, LocationRepository repo) {
        this.svc = svc;
        this.repo = repo;
    }

    @PostMapping
    public Location create(@RequestParam String name,
                           @RequestParam double lon,
                           @RequestParam double lat) {
        return svc.create(name, lon, lat);
    }

    @GetMapping("/nearby")
    public List<Location> nearby(@RequestParam double lon,
                                 @RequestParam double lat,
                                 @RequestParam(defaultValue = "1000") double meters) {
        return repo.findNearby(lon, lat, meters);
    }
}
```

---

### 9) How to test

1. Start Postgres and enable PostGIS:

   ```sql
   CREATE DATABASE geodb;
   \c geodb
   CREATE EXTENSION postgis;
   ```
2. Run Spring Boot app.
3. Insert a location:

   ```
   POST http://localhost:8080/api/locations?name=Delhi&lon=77.2090&lat=28.6139
   ```
4. Query nearby:

   ```
   GET http://localhost:8080/api/locations/nearby?lon=77.20&lat=28.61&meters=5000
   ```

---

👉 Do you want me to also include **sample curl/Postman requests + example JSON responses** for clarity, so you can verify quickly after running?

---

---
Perfect 👍 Let’s add **sample requests & responses** so you can test your Spring Boot + Hibernate Spatial project right away.

---

# 🔹 Sample API Test (Postman / curl)

### 1) Insert a location

**Request:**

```bash
curl -X POST "http://localhost:8080/api/locations?name=Delhi&lon=77.2090&lat=28.6139"
```

**Response (JSON):**

```json
{
  "id": 1,
  "name": "Delhi",
  "geom": {
    "type": "Point",
    "coordinates": [77.209, 28.6139]
  }
}
```

---

### 2) Insert another location

**Request:**

```bash
curl -X POST "http://localhost:8080/api/locations?name=Noida&lon=77.3910&lat=28.5355"
```

**Response:**

```json
{
  "id": 2,
  "name": "Noida",
  "geom": {
    "type": "Point",
    "coordinates": [77.391, 28.5355]
  }
}
```

---

### 3) Query nearby (5 km radius from Delhi)

**Request:**

```bash
curl "http://localhost:8080/api/locations/nearby?lon=77.20&lat=28.61&meters=5000"
```

**Response:**

```json
[
  {
    "id": 1,
    "name": "Delhi",
    "geom": {
      "type": "Point",
      "coordinates": [77.209, 28.6139]
    }
  },
  {
    "id": 2,
    "name": "Noida",
    "geom": {
      "type": "Point",
      "coordinates": [77.391, 28.5355]
    }
  }
]
```

👉 Both Delhi and Noida are within \~20 km, so depending on your `meters` value, they may show up.

---

### 4) Query nearby (1 km radius from Delhi)

**Request:**

```bash
curl "http://localhost:8080/api/locations/nearby?lon=77.20&lat=28.61&meters=1000"
```

**Response:**

```json
[
  {
    "id": 1,
    "name": "Delhi",
    "geom": {
      "type": "Point",
      "coordinates": [77.209, 28.6139]
    }
  }
]
```

Only Delhi is returned because Noida is further away.

---

✅ Now you have:

* Full Spring Boot + Hibernate Spatial project code
* PostGIS setup SQL
* Sample curl/Postman requests & responses

Would you like me to also include **SQL queries (direct PostGIS)** that match what the repository method does (so you can verify results at the database level too)?

---

---
Great 🚀 Let’s add **direct SQL queries** you can run in `psql` to verify what your Spring Boot app is doing under the hood.

This way, you can check PostGIS results independently.

---

# 🔹 SQL Queries (Direct PostGIS Verification)

### 1) Check that PostGIS is enabled

```sql
SELECT PostGIS_full_version();
```

✅ Should print something like:

```
POSTGIS="3.4.0" [EXTENSION] ...
```

---

### 2) Inspect the `locations` table

```sql
SELECT id, name, ST_AsText(geom) AS geom_wkt
FROM locations;
```

Example output:

```
 id |  name  |         geom_wkt
----+--------+----------------------------
  1 | Delhi  | POINT(77.209 28.6139)
  2 | Noida  | POINT(77.391 28.5355)
```

---

### 3) Find locations within **5 km** of Delhi (lon=77.20, lat=28.61)

```sql
SELECT id, name, ST_AsText(geom) AS geom_wkt
FROM locations
WHERE ST_DWithin(
        geom::geography,
        ST_SetSRID(ST_MakePoint(77.20, 28.61), 4326)::geography,
        5000  -- radius in meters
      );
```

Expected output:

```
 id |  name  |         geom_wkt
----+--------+----------------------------
  1 | Delhi  | POINT(77.209 28.6139)
  2 | Noida  | POINT(77.391 28.5355)
```

---

### 4) Find locations within **1 km** of Delhi

```sql
SELECT id, name, ST_AsText(geom) AS geom_wkt
FROM locations
WHERE ST_DWithin(
        geom::geography,
        ST_SetSRID(ST_MakePoint(77.20, 28.61), 4326)::geography,
        1000
      );
```

Expected output:

```
 id |  name  |         geom_wkt
----+--------+----------------------------
  1 | Delhi  | POINT(77.209 28.6139)
```

---

### 5) Measure actual distance between Delhi & Noida

```sql
SELECT ST_DistanceSphere(
         ST_SetSRID(ST_MakePoint(77.209, 28.6139), 4326),
         ST_SetSRID(ST_MakePoint(77.391, 28.5355), 4326)
       ) AS meters;
```

Example output:

```
 meters
---------
 20715.6
```

(\~20.7 km, so Noida won’t appear if you set a smaller radius like `1000`).

---

⚡ Now you can:

* Use **Spring Boot REST API** to add/query points, OR
* Use **SQL queries in Postgres** to directly validate spatial results.

---

👉 Do you also want me to add an **SQL insert script** (so you can preload multiple cities/points into PostGIS and test queries without calling the API repeatedly)?

---

---
Perfect 👍 Let’s make a **SQL insert script** with some Indian cities (Delhi, Noida, Gurugram, Agra, Jaipur) so you can preload your PostGIS `locations` table and test Hibernate Spatial queries right away.

---

# 🔹 SQL Insert Script (cities with geometry points)

```sql
-- Ensure PostGIS is enabled
CREATE EXTENSION IF NOT EXISTS postgis;

-- If table doesn’t exist, create it
CREATE TABLE IF NOT EXISTS locations (
    id SERIAL PRIMARY KEY,
    name TEXT,
    geom geometry(Point, 4326)
);

-- Insert cities (lon, lat coordinates with SRID 4326)
INSERT INTO locations (name, geom)
VALUES 
  ('Delhi',     ST_SetSRID(ST_MakePoint(77.2090, 28.6139), 4326)),
  ('Noida',     ST_SetSRID(ST_MakePoint(77.3910, 28.5355), 4326)),
  ('Gurugram',  ST_SetSRID(ST_MakePoint(77.0266, 28.4595), 4326)),
  ('Agra',      ST_SetSRID(ST_MakePoint(78.0081, 27.1767), 4326)),
  ('Jaipur',    ST_SetSRID(ST_MakePoint(75.7873, 26.9124), 4326));

-- Optional: spatial index for fast queries
CREATE INDEX IF NOT EXISTS idx_locations_geom
  ON locations
  USING GIST (geom);
```

---

# 🔹 Verify inserted data

```sql
SELECT id, name, ST_AsText(geom) AS geom_wkt
FROM locations;
```

Example output:

```
 id |   name    |         geom_wkt
----+-----------+----------------------------
  1 | Delhi     | POINT(77.209 28.6139)
  2 | Noida     | POINT(77.391 28.5355)
  3 | Gurugram  | POINT(77.0266 28.4595)
  4 | Agra      | POINT(78.0081 27.1767)
  5 | Jaipur    | POINT(75.7873 26.9124)
```

---

# 🔹 Example test queries

👉 Find cities within **50 km** of Delhi:

```sql
SELECT name
FROM locations
WHERE ST_DWithin(
        geom::geography,
        ST_SetSRID(ST_MakePoint(77.2090, 28.6139), 4326)::geography,
        50000
      );
```

Expected:

```
Delhi
Noida
Gurugram
```

👉 Distance Delhi → Agra:

```sql
SELECT ST_DistanceSphere(
         ST_SetSRID(ST_MakePoint(77.2090, 28.6139), 4326),
         ST_SetSRID(ST_MakePoint(78.0081, 27.1767), 4326)
       ) AS meters;
```

\~179,000 meters (\~179 km).

---

⚡ With this script you’ll have **real test data** for both PostGIS SQL and your **Spring Boot Hibernate Spatial APIs**.

---

Do you also want me to show you how to **visualize these points on a map (Leaflet / OpenLayers in a Boot web page)**, so you can actually see them instead of just JSON/SQL results?

---


---
Awesome 🌍✨ Let’s go one step further and **visualize your PostGIS + Hibernate Spatial data on a map** inside your Spring Boot project.
We’ll use **Leaflet.js** (lightweight, open-source, perfect for quick mapping).

---

# 🔹 Step 1: Create a REST endpoint returning GeoJSON

👉 Extend your controller so your `Location` data can be consumed by mapping libraries.

```java
package com.example.geo.controller;

import com.example.geo.entity.Location;
import com.example.geo.repo.LocationRepository;
import org.springframework.web.bind.annotation.*;

import java.util.*;

@RestController
@RequestMapping("/api/geojson")
public class GeoJsonController {

    private final LocationRepository repo;

    public GeoJsonController(LocationRepository repo) {
        this.repo = repo;
    }

    @GetMapping
    public Map<String, Object> getAllAsGeoJson() {
        List<Location> locations = repo.findAll();

        // Build GeoJSON FeatureCollection
        Map<String, Object> featureCollection = new LinkedHashMap<>();
        featureCollection.put("type", "FeatureCollection");

        List<Map<String, Object>> features = new ArrayList<>();

        for (Location loc : locations) {
            Map<String, Object> feature = new LinkedHashMap<>();
            feature.put("type", "Feature");

            // geometry
            Map<String, Object> geometry = new LinkedHashMap<>();
            geometry.put("type", "Point");
            geometry.put("coordinates", new double[]{
                    loc.getGeom().getX(), // lon
                    loc.getGeom().getY()  // lat
            });

            // properties
            Map<String, Object> properties = new LinkedHashMap<>();
            properties.put("id", loc.getId());
            properties.put("name", loc.getName());

            feature.put("geometry", geometry);
            feature.put("properties", properties);

            features.add(feature);
        }

        featureCollection.put("features", features);
        return featureCollection;
    }
}
```

✅ Now `GET http://localhost:8080/api/geojson` returns something like:

```json
{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "geometry": { "type": "Point", "coordinates": [77.209, 28.6139] },
      "properties": { "id": 1, "name": "Delhi" }
    },
    ...
  ]
}
```

---

# 🔹 Step 2: Add a simple web page with Leaflet

👉 Create `src/main/resources/static/map.html`

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8"/>
  <title>Locations Map</title>
  <link rel="stylesheet" href="https://unpkg.com/leaflet/dist/leaflet.css"/>
  <style>
    #map { height: 600px; width: 100%; }
  </style>
</head>
<body>
  <h2>📍 Locations from PostGIS</h2>
  <div id="map"></div>

  <script src="https://unpkg.com/leaflet/dist/leaflet.js"></script>
  <script>
    // Init map
    var map = L.map('map').setView([28.6, 77.2], 7); // center on Delhi

    // Add OpenStreetMap tiles
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
      attribution: '© OpenStreetMap contributors'
    }).addTo(map);

    // Fetch GeoJSON from backend
    fetch('/api/geojson')
      .then(res => res.json())
      .then(data => {
        L.geoJSON(data, {
          onEachFeature: function (feature, layer) {
            layer.bindPopup(`<b>${feature.properties.name}</b>`);
          }
        }).addTo(map);
      });
  </script>
</body>
</html>
```

---

# 🔹 Step 3: Run and Test

1. Start your Spring Boot app.
2. Open [http://localhost:8080/map.html](http://localhost:8080/map.html) in the browser.
3. ✅ You’ll see Delhi, Noida, Gurugram, Agra, Jaipur plotted on the map.
4. Click on a marker → popup with city name.

---

# 🔹 Optional Enhancements

* Style markers differently (`pointToLayer` with `L.circleMarker`).
* Add a **radius circle** for nearby queries (use `/api/locations/nearby`).
* Add polygons (e.g., state boundaries) with PostGIS `Polygon` geometries.

---

👉 Do you want me to also extend this example to **support polygons (e.g., draw a polygon in Leaflet and send it to Spring Boot → save in PostGIS)** so you can handle not just points but areas (boundaries, zones)?

---

---
Great 🚀 Let’s complete the flow with **Service + Controller + Testing Hibernate Spatial with PostGIS** in a Spring Boot app.

---

# ✅ Step 6: Service Layer

```java
package com.example.spatial.service;

import com.example.spatial.entity.City;
import com.example.spatial.repository.CityRepository;
import org.locationtech.jts.geom.Point;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class CityService {

    private final CityRepository cityRepository;

    public CityService(CityRepository cityRepository) {
        this.cityRepository = cityRepository;
    }

    // Save a city
    public City saveCity(String name, Point location) {
        City city = new City(name, location);
        return cityRepository.save(city);
    }

    // Find all cities
    public List<City> getAllCities() {
        return cityRepository.findAll();
    }

    // Find nearby cities using PostGIS function
    public List<City> findNearbyCities(Point location, double distanceInMeters) {
        return cityRepository.findNearbyCities(location, distanceInMeters);
    }
}
```

---

# ✅ Step 7: REST Controller

```java
package com.example.spatial.controller;

import com.example.spatial.entity.City;
import com.example.spatial.service.CityService;
import org.locationtech.jts.geom.GeometryFactory;
import org.locationtech.jts.geom.Point;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/cities")
public class CityController {

    private final CityService cityService;
    private final GeometryFactory geometryFactory = new GeometryFactory();

    public CityController(CityService cityService) {
        this.cityService = cityService;
    }

    // Add a new city
    @PostMapping("/add")
    public City addCity(@RequestParam String name,
                        @RequestParam double lat,
                        @RequestParam double lon) {
        Point location = geometryFactory.createPoint(new org.locationtech.jts.geom.Coordinate(lon, lat));
        return cityService.saveCity(name, location);
    }

    // Get all cities
    @GetMapping("/all")
    public List<City> getAllCities() {
        return cityService.getAllCities();
    }

    // Find nearby cities
    @GetMapping("/nearby")
    public List<City> findNearby(@RequestParam double lat,
                                 @RequestParam double lon,
                                 @RequestParam double distance) {
        Point location = geometryFactory.createPoint(new org.locationtech.jts.geom.Coordinate(lon, lat));
        return cityService.findNearbyCities(location, distance);
    }
}
```

---

# ✅ Step 8: Testing APIs

### 1. Insert Cities

```bash
POST http://localhost:8080/cities/add?name=Delhi&lat=28.7041&lon=77.1025
POST http://localhost:8080/cities/add?name=Noida&lat=28.5355&lon=77.3910
POST http://localhost:8080/cities/add?name=Gurgaon&lat=28.4595&lon=77.0266
```

### 2. Get All Cities

```bash
GET http://localhost:8080/cities/all
```

### 3. Find Nearby Cities (within 30 km of Delhi)

```bash
GET http://localhost:8080/cities/nearby?lat=28.7041&lon=77.1025&distance=30000
```

---

# ✅ Step 9: Verify SQL Generated by Hibernate

Hibernate will generate queries like:

```sql
SELECT * 
FROM city c
WHERE ST_DWithin(c.location, ST_SetSRID(ST_MakePoint(77.1025, 28.7041), 4326), 30000);
```

This uses **PostGIS spatial functions**.

---

👉 Now you have a **full Spring Boot + Hibernate Spatial + PostGIS project** with:

* Entity (`City`)
* Repository
* Service
* Controller
* REST APIs for geospatial queries

---

Do you also want me to **add Kafka Streams integration** with this (so spatial events like “new city added” go into Kafka)?

---


---
Nice — let’s wire Kafka *Streams* into your spatial app. Below I’ll give a compact, runnable integration you can drop into the project you already have:

What you’ll get

* Maven deps to add (Spring Kafka + Kafka Streams)
* `application.properties` entries for Kafka
* `CityEvent` DTO (JSON serializable)
* A simple **producer** that publishes a city event when you save a city
* A **Kafka Streams topology** that:

  * consumes city events from topic `city-events`
  * computes a running **count per city name** and writes counts to `city-counts`
  * filters incoming events by distance to a given point (example: Delhi center) using a Haversine function and writes those within X km to `nearby-city-events`
* Example consumer (listener) that logs processed results
* Sample curl/test steps

No unnecessary complexity — you can extend the topology later (e.g., enrich with PostGIS queries, join with other streams, windowed aggregations).

---

## 1) Add dependencies (pom.xml)

Add these dependencies inside your `<dependencies>`:

```xml
<!-- Spring for Apache Kafka (producer/consumer) -->
<dependency>
  <groupId>org.springframework.kafka</groupId>
  <artifactId>spring-kafka</artifactId>
</dependency>

<!-- Kafka Streams support -->
<dependency>
  <groupId>org.springframework.kafka</groupId>
  <artifactId>spring-kafka</artifactId>
  <classifier>streams</classifier>
  <scope>runtime</scope>
</dependency>

<!-- Jackson for JSON (if not already present) -->
<dependency>
  <groupId>com.fasterxml.jackson.core</groupId>
  <artifactId>jackson-databind</artifactId>
</dependency>
```

(If you prefer Gradle, let me know — I can provide the `build.gradle` lines.)

---

## 2) application.properties (Kafka config)

Add these Kafka properties (adjust `bootstrap.servers` to your Kafka):

```properties
# Kafka bootstrap (change to your broker host:port)
spring.kafka.bootstrap-servers=localhost:9092

# Kafka producer defaults (optional)
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.value-serializer=org.springframework.kafka.support.serializer.JsonSerializer

# Kafka consumer defaults (used by listeners)
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.springframework.kafka.support.serializer.JsonDeserializer
spring.kafka.consumer.properties.spring.json.trusted.packages=*

# Kafka Streams config prefix (Spring Boot auto config expects spring.kafka.streams)
spring.kafka.streams.application-id=geo-spatial-streams-app
spring.kafka.streams.bootstrap-servers=${spring.kafka.bootstrap-servers}
spring.kafka.streams.default.key.serde=org.apache.kafka.common.serialization.Serdes$StringSerde
spring.kafka.streams.default.value.serde=org.apache.kafka.common.serialization.Serdes$StringSerde
```

Notes:

* `spring.kafka.consumer.properties.spring.json.trusted.packages=*` lets Spring's JsonDeserializer accept DTOs from any package. Tighten for production.

---

## 3) DTO: CityEvent

Create a simple JSON-serializable DTO used as Kafka payload.

```java
package com.example.geo.kafka;

public class CityEvent {
    private Long id;
    private String name;
    private double lon;
    private double lat;

    public CityEvent() {}

    public CityEvent(Long id, String name, double lon, double lat) {
        this.id = id; this.name = name; this.lon = lon; this.lat = lat;
    }
    // getters & setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public double getLon() { return lon; }
    public void setLon(double lon) { this.lon = lon; }
    public double getLat() { return lat; }
    public void setLat(double lat) { this.lat = lat; }
}
```

---

## 4) Producer: publish event when city saved

Modify your `LocationService` (or `CityService`) to publish a `CityEvent` via `KafkaTemplate` after saving.

```java
package com.example.geo.service;

import com.example.geo.entity.Location;
import com.example.geo.kafka.CityEvent;
import com.example.geo.repo.LocationRepository;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.locationtech.jts.geom.Point;

@Service
public class LocationService {
    private final LocationRepository repo;
    private final KafkaTemplate<String, CityEvent> kafkaTemplate; // generic KafkaTemplate
    private final String topic = "city-events";

    public LocationService(LocationRepository repo, KafkaTemplate<String, CityEvent> kafkaTemplate) {
        this.repo = repo;
        this.kafkaTemplate = kafkaTemplate;
    }

    public Location create(String name, double lon, double lat) {
        Point p = new org.locationtech.jts.geom.GeometryFactory(new org.locationtech.jts.geom.PrecisionModel(), 4326)
                    .createPoint(new org.locationtech.jts.geom.Coordinate(lon, lat));
        Location loc = new Location();
        loc.setName(name);
        loc.setGeom(p);
        Location saved = repo.save(loc);

        // publish event
        CityEvent event = new CityEvent(saved.getId(), saved.getName(), saved.getGeom().getX(), saved.getGeom().getY());
        kafkaTemplate.send(topic, saved.getId().toString(), event);

        return saved;
    }
}
```

Also add a `KafkaTemplate` bean configuration (Spring Boot auto-config often provides one if `spring-kafka` on classpath and `spring.kafka` props present). If needed, add explicit bean:

```java
@Configuration
public class KafkaProducerConfig {
    @Bean
    public ProducerFactory<String, CityEvent> producerFactory(ObjectMapper mapper) {
        var props = new HashMap<String, Object>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        return new DefaultKafkaProducerFactory<>(props);
    }

    @Bean
    public KafkaTemplate<String, CityEvent> kafkaTemplate(ProducerFactory<String, CityEvent> pf) {
        return new KafkaTemplate<>(pf);
    }
}
```

---

## 5) Kafka Streams topology

Create a Streams configuration class that builds the topology described earlier.

```java
package com.example.geo.kafka;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.kafka.common.serialization.Serde;
import org.apache.kafka.common.serialization.Serdes;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.support.serializer.JsonSerde;
import org.springframework.kafka.annotation.EnableKafkaStreams;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.kstream.*;

@Configuration
@EnableKafkaStreams
public class StreamsConfig {

    // Haversine helper (meters)
    public static double haversine(double lat1, double lon1, double lat2, double lon2) {
        final int R = 6371000; // meters
        double dLat = Math.toRadians(lat2 - lat1);
        double dLon = Math.toRadians(lon2 - lon1);
        double a = Math.sin(dLat/2) * Math.sin(dLat/2)
                 + Math.cos(Math.toRadians(lat1)) * Math.cos(Math.toRadians(lat2))
                 * Math.sin(dLon/2) * Math.sin(dLon/2);
        double c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
        return R * c;
    }

    @Bean
    public KStream<String, CityEvent> kStream(StreamsBuilder builder) {
        final String inputTopic = "city-events";
        final String nearbyTopic = "nearby-city-events";
        final String countsTopic = "city-counts";

        // use JSON serde for CityEvent
        JsonSerde<CityEvent> cityEventSerde = new JsonSerde<>(CityEvent.class);

        KStream<String, CityEvent> stream = builder.stream(inputTopic,
                Consumed.with(Serdes.String(), cityEventSerde));

        // 1) count occurrences by city name (simple aggregation)
        stream
          .groupBy((key, event) -> event.getName(), Grouped.with(Serdes.String(), cityEventSerde))
          .count(Materialized.as("city-counts-store"))
          .toStream()
          .mapValues(Long::toString)
          .to(countsTopic, Produced.with(Serdes.String(), Serdes.String()));

        // 2) filter by distance to a fixed point (e.g., Delhi center)
        final double targetLat = 28.6139;
        final double targetLon = 77.2090;
        final double thresholdMeters = 50000; // 50 km

        stream.filter((k, event) -> {
            double d = haversine(targetLat, targetLon, event.getLat(), event.getLon());
            return d <= thresholdMeters;
        }).to(nearbyTopic, Produced.with(Serdes.String(), cityEventSerde));

        // return main stream for further chaining (or null)
        return stream;
    }
}
```

Explanation:

* `city-events` receives every saved city as JSON (published by the service).
* Topology groups events by `name` and produces running counts to `city-counts` (as string values).
* Also filters events within 50 km of Delhi and writes to `nearby-city-events`.

---

## 6) Example listener to log processed messages

You might want to observe output topics. Use a simple consumer/listener:

```java
package com.example.geo.kafka;

import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;

@Component
public class CityCountsListener {

    @KafkaListener(topics = "city-counts", groupId = "geo-loggers")
    public void handleCounts(String message) {
        System.out.println("city-counts msg: " + message);
    }

    @KafkaListener(topics = "nearby-city-events", groupId = "geo-loggers",
                   containerFactory = "kafkaListenerContainerFactory")
    public void handleNearby(CityEvent event) {
        System.out.println("nearby event: " + event.getName() + " @ " + event.getLat() + "," + event.getLon());
    }
}
```

If the `nearby-city-events` listener needs JSON → ensure listener container factory is configured to use `JsonDeserializer` for `CityEvent`. Spring Boot may auto-configure; if not, add a `ConcurrentKafkaListenerContainerFactory<String, CityEvent>` bean.

---

## 7) Create topics (locally)

If you run Kafka locally, create topics:

```bash
kafka-topics.sh --create --topic city-events --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1
kafka-topics.sh --create --topic city-counts --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1
kafka-topics.sh --create --topic nearby-city-events --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1
```

(If you use Confluent or other managed Kafka, create topics via their UI or CLI.)

---

## 8) Test flow (quick)

1. Start Kafka & Zookeeper.
2. Start your Spring Boot app (it will auto-create the Streams app).
3. Call API to add a city (example using your endpoint):

```bash
curl -X POST "http://localhost:8080/api/locations?name=Faridabad&lon=77.3050&lat=28.4089"
```

4. The `LocationService` will save it and publish to `city-events`.
5. Kafka Streams will consume it, update running counts to `city-counts`, and if within 50 km of Delhi, publish to `nearby-city-events`.
6. Watch the console logs from `CityCountsListener` — you should see outputs.

---

## 9) Next steps & ideas

* Replace the simple name-based aggregation with **spatial joins**: join the stream with data from a KTable containing polygons (zones) to find which zone a new city falls into. For that you might store polygon geometries in a compact serialized form and use a custom `ValueTransformer` with JTS to do `contains()` checks.
* Do **windowed aggregates** (counts per city per hour/day) using `.windowedBy(...)`.
* Use **Avro / Schema Registry** for typed events in production.
* Instead of Haversine in Streams, you can offload heavy spatial checks to a service that queries PostGIS (make a stream-processor call/out-of-band enrichment).

---

If you want, I can:

* provide the **full files** (complete Java classes with package statements) ready to paste into your project, or
* provide a **docker-compose** that launches Kafka + Zookeeper + Postgres+PostGIS for local end-to-end testing, or
* convert the Streams example from simple name-count to **spatial join** with polygon KTable and JTS-based `contains` checks.

Which of these next steps would you like?

---


---
Perfect 👍 Let’s now **summarize the full stepwise Hibernate Spatial with Spring Boot integration** (PostGIS + JTS).

---

# 📘 Stepwise Notes: Hibernate Spatial with Spring Boot + PostGIS

---

## **1. What is Hibernate Spatial?**

* An **extension of Hibernate ORM** to handle geospatial data.
* Provides **Java bindings for PostGIS** (PostgreSQL GIS extension).
* Works with **Geometry objects** (`Point`, `LineString`, `Polygon`).
* Uses **JTS (Java Topology Suite)** for geometry modeling.

---

## **2. Dependencies (Maven `pom.xml`)**

```xml
<dependencies>
    <!-- Spring Boot JPA -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <!-- PostgreSQL Driver -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>

    <!-- Hibernate Spatial (PostGIS support) -->
    <dependency>
        <groupId>org.hibernate</groupId>
        <artifactId>hibernate-spatial</artifactId>
        <version>6.4.4.Final</version> <!-- match Boot Hibernate version -->
    </dependency>

    <!-- JTS for geometry modeling -->
    <dependency>
        <groupId>org.locationtech.jts</groupId>
        <artifactId>jts-core</artifactId>
        <version>1.19.0</version>
    </dependency>
</dependencies>
```

---

## **3. Database Setup (PostgreSQL + PostGIS)**

1. Install PostgreSQL.
2. Enable PostGIS extension:

```sql
CREATE EXTENSION postgis;
```

3. Create schema/table with `geometry` columns.

---

## **4. `application.properties`**

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/spatialdb
spring.datasource.username=postgres
spring.datasource.password=postgres

spring.jpa.database-platform=org.hibernate.spatial.dialect.postgis.PostgisDialect
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
```

---

## **5. Entity Class with Spatial Column**

```java
import jakarta.persistence.*;
import org.locationtech.jts.geom.Point;

@Entity
@Table(name = "locations")
public class Location {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    // PostGIS geometry column
    @Column(columnDefinition = "geometry(Point, 4326)") 
    private Point coordinates;

    // Getters and Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }

    public Point getCoordinates() { return coordinates; }
    public void setCoordinates(Point coordinates) { this.coordinates = coordinates; }
}
```

---

## **6. Repository Layer**

```java
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.stereotype.Repository;
import java.util.List;

@Repository
public interface LocationRepository extends JpaRepository<Location, Long> {

    // Example: Find locations within radius using PostGIS
    @Query(value = "SELECT * FROM locations l WHERE ST_DWithin(l.coordinates, ST_SetSRID(ST_MakePoint(?1, ?2), 4326), ?3)", nativeQuery = true)
    List<Location> findWithinRadius(double lon, double lat, double radiusMeters);
}
```

---

## **7. Service Layer**

```java
import org.locationtech.jts.geom.*;
import org.springframework.stereotype.Service;
import java.util.List;

@Service
public class LocationService {
    private final LocationRepository repo;
    private final GeometryFactory geometryFactory = new GeometryFactory();

    public LocationService(LocationRepository repo) {
        this.repo = repo;
    }

    public Location saveLocation(String name, double lon, double lat) {
        Point point = geometryFactory.createPoint(new Coordinate(lon, lat));
        point.setSRID(4326); // WGS84
        Location loc = new Location();
        loc.setName(name);
        loc.setCoordinates(point);
        return repo.save(loc);
    }

    public List<Location> getNearby(double lon, double lat, double radiusMeters) {
        return repo.findWithinRadius(lon, lat, radiusMeters);
    }
}
```

---

## **8. REST Controller**

```java
import org.springframework.web.bind.annotation.*;
import java.util.List;

@RestController
@RequestMapping("/locations")
public class LocationController {
    private final LocationService service;

    public LocationController(LocationService service) {
        this.service = service;
    }

    @PostMapping
    public Location addLocation(@RequestParam String name, @RequestParam double lon, @RequestParam double lat) {
        return service.saveLocation(name, lon, lat);
    }

    @GetMapping("/nearby")
    public List<Location> findNearby(@RequestParam double lon, @RequestParam double lat, @RequestParam double radius) {
        return service.getNearby(lon, lat, radius);
    }
}
```

---

## **9. Example Usage**

* **Add a Location**

```http
POST /locations?name=Delhi&lon=77.2090&lat=28.6139
```

* **Find Nearby Locations**

```http
GET /locations/nearby?lon=77.2&lat=28.61&radius=5000
```

---

## **10. Real-World Uses**

* **Ride-hailing apps** → Find nearby drivers.
* **Logistics** → Track delivery points inside a polygon region.
* **Geofencing** → Alerts when an object enters/leaves a region.
* **Mapping apps** → Store & query map features.

---

✅ With this setup, your Boot project can fully handle **PostGIS + Hibernate Spatial** queries.
Do you want me to also prepare a **Topology use-case (Polygon geofencing example)** with code along with `ST_Contains` query?

---


---
Perfect 👍 Now let’s finalize with **stepwise usage of Hibernate Spatial in Boot with PostGIS**, with complete code:

---

## 📝 Stepwise Notes: Hibernate Spatial with Spring Boot + PostGIS

---

### **1. Add Dependencies**

Add Hibernate Spatial + PostGIS driver + JTS.

```xml
<dependencies>
    <!-- Spring Boot JPA -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <!-- PostgreSQL Driver -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
    </dependency>

    <!-- Hibernate Spatial -->
    <dependency>
        <groupId>org.hibernate</groupId>
        <artifactId>hibernate-spatial</artifactId>
    </dependency>

    <!-- JTS (Geometry library) -->
    <dependency>
        <groupId>org.locationtech.jts</groupId>
        <artifactId>jts-core</artifactId>
        <version>1.18.2</version>
    </dependency>
</dependencies>
```

---

### **2. Enable PostGIS Extension in PostgreSQL**

Login to DB and run:

```sql
CREATE EXTENSION IF NOT EXISTS postgis;
```

---

### **3. Configure `application.yml`**

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/spatialdb
    username: postgres
    password: root
  jpa:
    hibernate:
      ddl-auto: update
    properties:
      hibernate:
        dialect: org.hibernate.spatial.dialect.postgis.PostgisDialect
```

---

### **4. Create Entity with Geometry**

```java
import jakarta.persistence.*;
import org.locationtech.jts.geom.Point;

@Entity
@Table(name = "locations")
public class Location {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    // Spatial column
    @Column(columnDefinition = "Geometry(Point, 4326)")
    private Point coordinates;

    public Location() {}

    public Location(String name, Point coordinates) {
        this.name = name;
        this.coordinates = coordinates;
    }

    // Getters & Setters
}
```

---

### **5. Repository with Spatial Query**

```java
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import java.util.List;

public interface LocationRepository extends JpaRepository<Location, Long> {

    // Find all locations within a distance (in meters)
    @Query(value = "SELECT * FROM locations l WHERE ST_DWithin(l.coordinates, ST_SetSRID(ST_MakePoint(:lon, :lat), 4326), :distance)", 
           nativeQuery = true)
    List<Location> findNearby(@Param("lon") double lon,
                              @Param("lat") double lat,
                              @Param("distance") double distance);
}
```

---

### **6. Service Layer**

```java
import org.locationtech.jts.geom.GeometryFactory;
import org.locationtech.jts.geom.Point;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class LocationService {

    private final LocationRepository locationRepository;
    private final GeometryFactory geometryFactory = new GeometryFactory();

    public LocationService(LocationRepository locationRepository) {
        this.locationRepository = locationRepository;
    }

    public Location addLocation(String name, double lat, double lon) {
        Point point = geometryFactory.createPoint(new org.locationtech.jts.geom.Coordinate(lon, lat));
        point.setSRID(4326);
        return locationRepository.save(new Location(name, point));
    }

    public List<Location> findNearby(double lat, double lon, double distance) {
        return locationRepository.findNearby(lon, lat, distance);
    }
}
```

---

### **7. REST Controller**

```java
import org.springframework.web.bind.annotation.*;
import java.util.List;

@RestController
@RequestMapping("/api/locations")
public class LocationController {

    private final LocationService locationService;

    public LocationController(LocationService locationService) {
        this.locationService = locationService;
    }

    @PostMapping
    public Location addLocation(@RequestParam String name,
                                @RequestParam double lat,
                                @RequestParam double lon) {
        return locationService.addLocation(name, lat, lon);
    }

    @GetMapping("/nearby")
    public List<Location> getNearby(@RequestParam double lat,
                                    @RequestParam double lon,
                                    @RequestParam double distance) {
        return locationService.findNearby(lat, lon, distance);
    }
}
```

---

### **8. Test API Calls**

* **Add a Location**

```bash
POST http://localhost:8080/api/locations?name=Delhi&lat=28.7041&lon=77.1025
```

* **Find Nearby Locations (within 50 km)**

```bash
GET http://localhost:8080/api/locations/nearby?lat=28.70&lon=77.10&distance=50000
```

---

✅ Now your Spring Boot app supports **Hibernate Spatial + PostGIS** for geospatial queries.
You can store `Point`, `LineString`, `Polygon` and run queries like `ST_Within`, `ST_Distance`, `ST_Contains`.

---

Do you also want me to add **LineString & Polygon examples** (routes, city boundaries) along with queries like “find all points inside polygon”?

---

---
Perfect 👍 Let’s finalize your **Hibernate Spatial + Spring Boot + PostGIS** notes with **end-to-end working code**.

---

# 📌 Hibernate Spatial with Spring Boot & PostGIS – Stepwise Notes

---

## 1️⃣ What is Hibernate Spatial?

* Extension of Hibernate ORM for **geospatial data types**.
* Provides Java bindings for **PostGIS functions**.
* Works with geometries like:

  * `Point` (latitude/longitude)
  * `LineString` (roads, rivers)
  * `Polygon` (boundaries, areas)
* Uses **JTS (Java Topology Suite)** for geometry modeling.

---

## 2️⃣ Dependencies (`pom.xml`)

```xml
<dependencies>
    <!-- Spring Boot JPA -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <!-- PostgreSQL Driver -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>

    <!-- Hibernate Spatial -->
    <dependency>
        <groupId>org.hibernate</groupId>
        <artifactId>hibernate-spatial</artifactId>
    </dependency>

    <!-- JTS for Geometry Objects -->
    <dependency>
        <groupId>org.locationtech.jts</groupId>
        <artifactId>jts-core</artifactId>
        <version>1.19.0</version>
    </dependency>
</dependencies>
```

---

## 3️⃣ Enable PostGIS in Database

Run in PostgreSQL:

```sql
CREATE EXTENSION IF NOT EXISTS postgis;
```

---

## 4️⃣ Application Properties (`application.yml`)

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/geo_db
    username: postgres
    password: postgres
  jpa:
    hibernate:
      ddl-auto: update
    properties:
      hibernate:
        dialect: org.hibernate.spatial.dialect.postgis.PostgisDialect
```

---

## 5️⃣ Entity with Geometry Field

```java
import jakarta.persistence.*;
import org.locationtech.jts.geom.Point;
import lombok.Getter;
import lombok.Setter;

@Entity
@Getter @Setter
public class Location {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    // Geometry type from JTS
    @Column(columnDefinition = "geometry(Point,4326)")
    private Point coordinates;
}
```

---

## 6️⃣ Repository

```java
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface LocationRepository extends JpaRepository<Location, Long> {
}
```

---

## 7️⃣ Service Layer

```java
import org.locationtech.jts.geom.Coordinate;
import org.locationtech.jts.geom.GeometryFactory;
import org.springframework.stereotype.Service;

@Service
public class LocationService {

    private final LocationRepository repo;
    private final GeometryFactory geometryFactory = new GeometryFactory();

    public LocationService(LocationRepository repo) {
        this.repo = repo;
    }

    public Location saveLocation(String name, double lat, double lon) {
        Location loc = new Location();
        loc.setName(name);
        loc.setCoordinates(geometryFactory.createPoint(new Coordinate(lon, lat)));
        return repo.save(loc);
    }
}
```

---

## 8️⃣ Controller

```java
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/locations")
public class LocationController {

    private final LocationService service;
    private final LocationRepository repo;

    public LocationController(LocationService service, LocationRepository repo) {
        this.service = service;
        this.repo = repo;
    }

    @PostMapping
    public Location add(@RequestParam String name,
                        @RequestParam double lat,
                        @RequestParam double lon) {
        return service.saveLocation(name, lat, lon);
    }

    @GetMapping
    public List<Location> all() {
        return repo.findAll();
    }
}
```

---

## 9️⃣ Testing the API

Start Boot app, then:

👉 Save a new location

```bash
curl -X POST "http://localhost:8080/locations?name=Delhi&lat=28.7041&lon=77.1025"
```

👉 Fetch all

```bash
curl http://localhost:8080/locations
```

Response:

```json
[
  {
    "id": 1,
    "name": "Delhi",
    "coordinates": "POINT (77.1025 28.7041)"
  }
]
```

---

## 🔟 Extra: Spatial Queries (Custom Repo Example)

Find locations within a radius:

```java
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.util.List;

public interface LocationRepository extends JpaRepository<Location, Long> {

    @Query(value = "SELECT * FROM location " +
                   "WHERE ST_DWithin(coordinates, ST_MakePoint(:lon, :lat)::geography, :radius)",
           nativeQuery = true)
    List<Location> findNearby(@Param("lat") double lat,
                              @Param("lon") double lon,
                              @Param("radius") double radius);
}
```

---

✅ Now you have a **full working Spring Boot + Hibernate Spatial + PostGIS project** with:

* Geometry mapping
* REST APIs
* Custom geospatial queries

---

Do you also want me to add **Kafka Streams integration on top of this (for real-time geospatial data processing)** so it fits your IoT + PostGIS use case?

---

---
Nice — below is a complete, ready-to-drop-into-your-project **Kafka + Kafka Streams** integration for your Spring Boot + Hibernate Spatial application (`com.example.geo` package). It includes:

* Maven deps to add
* `application.yml` config
* DTO (`CityEvent`)
* Producer wired into your `LocationService`
* Streams topology (counts + nearby filter)
* Simple Kafka listeners to observe outputs
* `docker-compose.yml` to run Kafka (Zookeeper) + Postgres+PostGIS locally
* Topic creation commands and quick test steps

Drop these files into your project, start the stack, and you’ll have end-to-end flow: `POST /api/locations` → saved in PostGIS → `city-events` published → Kafka Streams processes → outputs to `city-counts` and `nearby-city-events` → listeners log results.

---

# 1) POM dependencies (add)

Add to your `pom.xml` inside `<dependencies>`:

```xml
<!-- Spring Kafka (producer/consumer) -->
<dependency>
  <groupId>org.springframework.kafka</groupId>
  <artifactId>spring-kafka</artifactId>
</dependency>

<!-- Spring Kafka Streams integration -->
<dependency>
  <groupId>org.springframework.kafka</groupId>
  <artifactId>spring-kafka-streams</artifactId>
</dependency>

<!-- Apache Kafka Streams core (runtime) -->
<dependency>
  <groupId>org.apache.kafka</groupId>
  <artifactId>kafka-streams</artifactId>
</dependency>

<!-- Jackson JSON handling -->
<dependency>
  <groupId>com.fasterxml.jackson.core</groupId>
  <artifactId>jackson-databind</artifactId>
</dependency>
```

(You already have Spring Web, JPA, hibernate-spatial and JTS from earlier steps.)

---

# 2) application.yml

Add Kafka + Streams config (adjust bootstrap servers if not using docker-compose below):

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
    consumer:
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: "*"
    streams:
      application-id: geo-spatial-streams-app
      bootstrap-servers: ${spring.kafka.bootstrap-servers}
      properties:
        commit.interval.ms: 1000
      default:
        key:
          serde: org.apache.kafka.common.serialization.Serdes$StringSerde
        value:
          serde: org.apache.kafka.common.serialization.Serdes$StringSerde

# spring.datasource and jpa configuration (example)
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/geodb
    username: postgres
    password: postgres
  jpa:
    hibernate:
      ddl-auto: update
    properties:
      hibernate:
        dialect: org.hibernate.spatial.dialect.postgis.PostgisDialect
```

---

# 3) DTO — `CityEvent.java`

Create `src/main/java/com/example/geo/kafka/CityEvent.java`:

```java
package com.example.geo.kafka;

public class CityEvent {
    private Long id;
    private String name;
    private double lon;
    private double lat;

    public CityEvent() {}

    public CityEvent(Long id, String name, double lon, double lat) {
        this.id = id;
        this.name = name;
        this.lon = lon;
        this.lat = lat;
    }

    // getters & setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }

    public double getLon() { return lon; }
    public void setLon(double lon) { this.lon = lon; }

    public double getLat() { return lat; }
    public void setLat(double lat) { this.lat = lat; }
}
```

---

# 4) Kafka Producer config (optional — Spring Boot auto config often suffices)

If you want explicit beans, add `KafkaProducerConfig.java`:

```java
package com.example.geo.kafka;

import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.common.serialization.StringSerializer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.core.*;

import org.springframework.kafka.support.serializer.JsonSerializer;

import java.util.HashMap;
import java.util.Map;

@Configuration
public class KafkaProducerConfig {

    @Bean
    public ProducerFactory<String, CityEvent> producerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        return new DefaultKafkaProducerFactory<>(props);
    }

    @Bean
    public KafkaTemplate<String, CityEvent> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
}
```

---

# 5) Publish city events when saving (modify `LocationService`)

Replace or extend your existing service to publish after save. Example `LocationService.java`:

```java
package com.example.geo.service;

import com.example.geo.entity.Location;
import com.example.geo.kafka.CityEvent;
import com.example.geo.repo.LocationRepository;
import org.locationtech.jts.geom.GeometryFactory;
import org.locationtech.jts.geom.Point;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class LocationService {

    private final LocationRepository repo;
    private final KafkaTemplate<String, CityEvent> kafkaTemplate;
    private final GeometryFactory gf = new GeometryFactory();

    private final String topic = "city-events";

    public LocationService(LocationRepository repo, KafkaTemplate<String, CityEvent> kafkaTemplate) {
        this.repo = repo;
        this.kafkaTemplate = kafkaTemplate;
    }

    @Transactional
    public Location create(String name, double lon, double lat) {
        Point p = gf.createPoint(new org.locationtech.jts.geom.Coordinate(lon, lat));
        p.setSRID(4326);
        Location loc = new Location();
        loc.setName(name);
        loc.setCoordinates(p);
        Location saved = repo.save(loc);

        // publish event
        CityEvent event = new CityEvent(saved.getId(), saved.getName(), saved.getCoordinates().getX(), saved.getCoordinates().getY());
        kafkaTemplate.send(topic, saved.getId().toString(), event);

        return saved;
    }
}
```

---

# 6) Kafka Streams topology — `StreamsConfig.java`

`src/main/java/com/example/geo/kafka/StreamsConfig.java`:

```java
package com.example.geo.kafka;

import org.apache.kafka.common.serialization.Serde;
import org.apache.kafka.common.serialization.Serdes;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.annotation.EnableKafkaStreams;
import org.springframework.kafka.support.serializer.JsonSerde;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.kstream.*;

@Configuration
@EnableKafkaStreams
public class StreamsConfig {

    // Haversine distance in meters
    private static double haversine(double lat1, double lon1, double lat2, double lon2) {
        final int R = 6371000; // meters
        double dLat = Math.toRadians(lat2 - lat1);
        double dLon = Math.toRadians(lon2 - lon1);
        double a = Math.sin(dLat/2) * Math.sin(dLat/2)
                 + Math.cos(Math.toRadians(lat1)) * Math.cos(Math.toRadians(lat2))
                 * Math.sin(dLon/2) * Math.sin(dLon/2);
        double c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
        return R * c;
    }

    @Bean
    public KStream<String, CityEvent> kStream(StreamsBuilder builder) {
        final String inputTopic = "city-events";
        final String nearbyTopic = "nearby-city-events";
        final String countsTopic = "city-counts";

        JsonSerde<CityEvent> citySerde = new JsonSerde<>(CityEvent.class);
        Serde<String> stringSerde = Serdes.String();

        KStream<String, CityEvent> stream = builder.stream(inputTopic, Consumed.with(stringSerde, citySerde));

        // 1) count occurrences by city name
        stream.groupBy((key, event) -> event.getName(), Grouped.with(stringSerde, citySerde))
              .count(Materialized.as("city-counts-store"))
              .toStream()
              .mapValues(Object::toString)
              .to(countsTopic, Produced.with(stringSerde, stringSerde));

        // 2) filter by distance to a target point (Delhi center as example)
        final double targetLat = 28.6139;
        final double targetLon = 77.2090;
        final double thresholdMeters = 50000; // 50 km

        stream.filter((k, event) -> {
            double d = haversine(targetLat, targetLon, event.getLat(), event.getLon());
            return d <= thresholdMeters;
        }).to(nearbyTopic, Produced.with(stringSerde, citySerde));

        return stream;
    }
}
```

---

# 7) Listeners to observe outputs — `CityListeners.java`

Create `src/main/java/com/example/geo/kafka/CityListeners.java`:

```java
package com.example.geo.kafka;

import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;

@Component
public class CityListeners {

    @KafkaListener(topics = "city-counts", groupId = "geo-loggers")
    public void onCityCounts(String message) {
        System.out.println("[city-counts] " + message);
    }

    // For JSON CityEvent on nearby-city-events (requires proper container factory auto-configured)
    @KafkaListener(topics = "nearby-city-events", groupId = "geo-loggers")
    public void onNearbyCityEvent(CityEvent event) {
        System.out.println("[nearby-city-events] " + event.getName() + " at " + event.getLat() + "," + event.getLon());
    }
}
```

> Note: Spring Boot often auto-configures listener container factories. If the `nearby-city-events` listener receives raw JSON strings instead of `CityEvent`, you can define a `ConcurrentKafkaListenerContainerFactory<String, CityEvent>` bean wired with `JsonDeserializer`.

---

# 8) Create topics (docker-compose or local Kafka)

If you use the docker-compose below, exec into the Kafka container to create topics, or use a Kafka client:

```bash
# If Kafka is on localhost:9092
kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic city-events
kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic city-counts
kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic nearby-city-events
```

---

# 9) docker-compose for local testing

Create `docker-compose.yml` at project root (quick local stack: Zookeeper, Kafka, Postgres+PostGIS):

```yaml
version: '3.8'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"

  kafka:
    image: confluentinc/cp-kafka:7.4.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

  postgres:
    image: postgis/postgis:15-3.4
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: geodb
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

Run:

```bash
docker compose up -d
```

After Postgres is up, enable PostGIS (connect with `psql` or any client):

```sql
CREATE EXTENSION IF NOT EXISTS postgis;
```

Create Kafka topics (enter kafka container or use host tools):

```bash
# inside kafka container (example)
docker exec -it <kafka-container-id> bash
kafka-topics --bootstrap-server localhost:9092 --create --topic city-events --partitions 1 --replication-factor 1
kafka-topics --bootstrap-server localhost:9092 --create --topic city-counts --partitions 1 --replication-factor 1
kafka-topics --bootstrap-server localhost:9092 --create --topic nearby-city-events --partitions 1 --replication-factor 1
```

---

# 10) Quick test flow

1. `docker compose up -d` (starts Kafka + Postgres).
2. Start your Spring Boot app (it will connect to Postgres & Kafka).
3. Create table / enable PostGIS (if not created by JPA): `CREATE EXTENSION postgis;` and optionally run insert SQL.
4. POST a new location via your REST endpoint:

```bash
curl -X POST "http://localhost:8080/api/locations?name=Faridabad&lon=77.3050&lat=28.4089"
```

5. Watch logs — you should see:

* message published to `city-events` (producer send)
* Kafka Streams consume & emit to `city-counts` and (if within threshold) `nearby-city-events`
* `CityListeners` prints results to console

6. Verify topics via `kafka-console-consumer` if you want to inspect raw messages.

---

# Notes, tips & next steps

* For **production** use Avro + Schema Registry or strong typed Serdes instead of raw JsonSerde.
* For **spatial-heavy** checks, you can replace the Haversine filter in Streams with an enrichment step that calls a tile service or queries PostGIS via a state store / external service (beware latency).
* To perform spatial **joins** (e.g., events → polygons/zones), pre-load polygon geometries into a KTable (topic with serialized polygons) and use a custom `ValueTransformer` with JTS `contains()` to output matched zone events. I can give code for that if you want.
* If listeners get raw JSON, add a `ConcurrentKafkaListenerContainerFactory` with `JsonDeserializer<CityEvent>` and set `spring.json.trusted.packages`.

---

If you want I can now:

* generate the **exact Java files** (full classes with imports) ready to paste; **or**
* provide the **polygon KTable spatial join** example (code + how to seed the polygon topic); **or**
* tailor the docker-compose to include **Schema Registry / Kafka UI** for easier testing.

Which of those next steps would you like?

---

---
Nice — I took your “yes” to mean **generate the exact Java files** (complete classes with imports and package statements) you can paste into your project right away.
I included the Kafka/Kafka-Streams wiring plus a robust listener factory so JSON `CityEvent` payloads deserialize into objects correctly.

Drop these files into your project under the package `com.example.geo` (subpackages as shown). Adjust `bootstrap.servers` in `application.yml` if needed.

---

## 1) `src/main/java/com/example/geo/kafka/CityEvent.java`

```java
package com.example.geo.kafka;

public class CityEvent {
    private Long id;
    private String name;
    private double lon;
    private double lat;

    public CityEvent() {}

    public CityEvent(Long id, String name, double lon, double lat) {
        this.id = id;
        this.name = name;
        this.lon = lon;
        this.lat = lat;
    }

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }

    public double getLon() { return lon; }
    public void setLon(double lon) { this.lon = lon; }

    public double getLat() { return lat; }
    public void setLat(double lat) { this.lat = lat; }

    @Override
    public String toString() {
        return "CityEvent{" +
                "id=" + id +
                ", name='" + name + '\'' +
                ", lon=" + lon +
                ", lat=" + lat +
                '}';
    }
}
```

---

## 2) `src/main/java/com/example/geo/kafka/KafkaProducerConfig.java`

```java
package com.example.geo.kafka;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.common.serialization.StringSerializer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.core.*;
import org.springframework.kafka.support.serializer.JsonSerializer;

import java.util.HashMap;
import java.util.Map;

@Configuration
public class KafkaProducerConfig {

    @Bean
    public ProducerFactory<String, CityEvent> producerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092"); // change if needed
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        // optional JsonSerializer config
        props.put(JsonSerializer.ADD_TYPE_INFO_HEADERS, false);
        return new DefaultKafkaProducerFactory<>(props);
    }

    @Bean
    public KafkaTemplate<String, CityEvent> kafkaTemplate(ProducerFactory<String, CityEvent> pf) {
        return new KafkaTemplate<>(pf);
    }
}
```

---

## 3) `src/main/java/com/example/geo/kafka/KafkaConsumerConfig.java`

```java
package com.example.geo.kafka;

import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.core.*;
import org.springframework.kafka.support.serializer.JsonDeserializer;

import java.util.HashMap;
import java.util.Map;

@Configuration
public class KafkaConsumerConfig {

    @Bean
    public ConsumerFactory<String, CityEvent> cityEventConsumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092"); // change if needed
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "geo-loggers");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);

        JsonDeserializer<CityEvent> deserializer = new JsonDeserializer<>(CityEvent.class);
        deserializer.addTrustedPackages("*");
        // disable type headers (if producer doesn't add them)
        deserializer.setUseTypeMapperForKey(false);

        return new DefaultKafkaConsumerFactory<>(props, new StringDeserializer(), deserializer);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, CityEvent> cityEventKafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, CityEvent> factory =
                new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(cityEventConsumerFactory());
        return factory;
    }
}
```

---

## 4) `src/main/java/com/example/geo/kafka/StreamsConfig.java`

```java
package com.example.geo.kafka;

import org.apache.kafka.common.serialization.Serde;
import org.apache.kafka.common.serialization.Serdes;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.annotation.EnableKafkaStreams;
import org.springframework.kafka.support.serializer.JsonSerde;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.kstream.*;

@Configuration
@EnableKafkaStreams
public class StreamsConfig {

    // Haversine distance (meters)
    private static double haversine(double lat1, double lon1, double lat2, double lon2) {
        final int R = 6371000; // meters
        double dLat = Math.toRadians(lat2 - lat1);
        double dLon = Math.toRadians(lon2 - lon1);
        double a = Math.sin(dLat/2) * Math.sin(dLat/2)
                 + Math.cos(Math.toRadians(lat1)) * Math.cos(Math.toRadians(lat2))
                 * Math.sin(dLon/2) * Math.sin(dLon/2);
        double c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
        return R * c;
    }

    @Bean
    public KStream<String, CityEvent> kStream(StreamsBuilder builder) {
        final String inputTopic = "city-events";
        final String nearbyTopic = "nearby-city-events";
        final String countsTopic = "city-counts";

        JsonSerde<CityEvent> citySerde = new JsonSerde<>(CityEvent.class);
        Serde<String> stringSerde = Serdes.String();

        KStream<String, CityEvent> stream = builder.stream(inputTopic, Consumed.with(stringSerde, citySerde));

        // 1) count by city name
        stream.groupBy((k, v) -> v.getName(), Grouped.with(stringSerde, citySerde))
              .count(Materialized.as("city-counts-store"))
              .toStream()
              .mapValues(Object::toString)
              .to(countsTopic, Produced.with(stringSerde, stringSerde));

        // 2) filter by distance to target (example: Delhi)
        final double targetLat = 28.6139;
        final double targetLon = 77.2090;
        final double thresholdMeters = 50000; // 50 km

        stream.filter((k, v) -> {
            double d = haversine(targetLat, targetLon, v.getLat(), v.getLon());
            return d <= thresholdMeters;
        }).to(nearbyTopic, Produced.with(stringSerde, citySerde));

        return stream;
    }
}
```

---

## 5) `src/main/java/com/example/geo/kafka/CityListeners.java`

```java
package com.example.geo.kafka;

import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;

@Component
public class CityListeners {

    // Listen to counts topic (string messages)
    @KafkaListener(topics = "city-counts", groupId = "geo-loggers")
    public void onCityCounts(String message) {
        System.out.println("[city-counts] " + message);
    }

    // Listen to nearby events and receive CityEvent objects
    @KafkaListener(topics = "nearby-city-events", groupId = "geo-loggers",
                   containerFactory = "cityEventKafkaListenerContainerFactory")
    public void onNearbyCityEvent(CityEvent event) {
        System.out.println("[nearby-city-events] " + event);
    }
}
```

---

## 6) `src/main/java/com/example/geo/service/LocationService.java`

*(Service that saves location to DB and publishes CityEvent)*

```java
package com.example.geo.service;

import com.example.geo.entity.Location;
import com.example.geo.kafka.CityEvent;
import com.example.geo.repo.LocationRepository;
import org.locationtech.jts.geom.*;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class LocationService {

    private final LocationRepository repo;
    private final KafkaTemplate<String, CityEvent> kafkaTemplate;
    private final GeometryFactory gf = new GeometryFactory(new PrecisionModel(), 4326);
    private final String topic = "city-events";

    public LocationService(LocationRepository repo, KafkaTemplate<String, CityEvent> kafkaTemplate) {
        this.repo = repo;
        this.kafkaTemplate = kafkaTemplate;
    }

    @Transactional
    public Location create(String name, double lon, double lat) {
        Point p = gf.createPoint(new Coordinate(lon, lat));
        p.setSRID(4326);
        Location loc = new Location();
        loc.setName(name);
        loc.setCoordinates(p);
        Location saved = repo.save(loc);

        CityEvent event = new CityEvent(saved.getId(), saved.getName(), saved.getCoordinates().getX(), saved.getCoordinates().getY());
        kafkaTemplate.send(topic, saved.getId().toString(), event);
        return saved;
    }
}
```

---

## 7) `src/main/java/com/example/geo/entity/Location.java`

*(if you don’t already have this exact entity — use this one)*

```java
package com.example.geo.entity;

import jakarta.persistence.*;
import org.locationtech.jts.geom.Point;

@Entity
@Table(name = "locations")
public class Location {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @Column(columnDefinition = "geometry(Point,4326)")
    private Point coordinates;

    public Location() {}

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }

    public Point getCoordinates() { return coordinates; }
    public void setCoordinates(Point coordinates) { this.coordinates = coordinates; }
}
```

---

## 8) `src/main/java/com/example/geo/repo/LocationRepository.java`

```java
package com.example.geo.repo;

import com.example.geo.entity.Location;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface LocationRepository extends JpaRepository<Location, Long> {
}
```

---

## 9) `src/main/java/com/example/geo/controller/LocationController.java`

```java
package com.example.geo.controller;

import com.example.geo.entity.Location;
import com.example.geo.repo.LocationRepository;
import com.example.geo.service.LocationService;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/locations")
public class LocationController {

    private final LocationService service;
    private final LocationRepository repo;

    public LocationController(LocationService service, LocationRepository repo) {
        this.service = service;
        this.repo = repo;
    }

    @PostMapping
    public Location create(@RequestParam String name,
                           @RequestParam double lon,
                           @RequestParam double lat) {
        return service.create(name, lon, lat);
    }

    @GetMapping
    public List<Location> all() {
        return repo.findAll();
    }
}
```

---

### Quick checklist to run end-to-end

1. Ensure your `pom.xml` includes `spring-kafka`, `spring-kafka-streams`, `kafka-streams`, `jackson-databind` and the Postgres/PostGIS + hibernate-spatial + JTS deps from earlier.
2. `application.yml` or `application.properties` must set `spring.kafka.bootstrap-servers=localhost:9092` and streams `application-id`.
3. Start Docker compose (or local services) so Kafka & Postgres+PostGIS are available (I provided a `docker-compose.yml` previously).
4. Create topics `city-events`, `city-counts`, `nearby-city-events`.
5. Start Spring Boot.
6. POST to your endpoint:

   ```
   curl -X POST "http://localhost:8080/api/locations?name=Faridabad&lon=77.3050&lat=28.4089"
   ```
7. Watch the Spring Boot logs — you should see `CityListeners` output for `city-counts` and `nearby-city-events` (if within threshold).

---

If you want next, I can:

* add the **polygon KTable spatial-join example** (seed polygon topic, Streams `ValueTransformer` using JTS `contains()` to detect which polygon a point falls into), **OR**
* provide the **docker-compose** again including Kafka UI / Schema Registry + scripts to auto-create topics, **OR**
* create a small **readme** with exact commands to run the stack and test end-to-end.

Pick one and I’ll generate it now.

---


-----[12]: https://central.sonatype.com/artifact/org.locationtech.jts/jts/1.18.2?utm_source=chatgpt.com "org.locationtech.jts:jts:1.18.2 - Maven Central"
[13]: https://www.baeldung.com/hibernate-spatial?utm_source=chatgpt.com "Introduction to Hibernate Spatial"
-
