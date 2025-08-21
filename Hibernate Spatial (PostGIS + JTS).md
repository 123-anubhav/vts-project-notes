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
[12]: https://central.sonatype.com/artifact/org.locationtech.jts/jts/1.18.2?utm_source=chatgpt.com "org.locationtech.jts:jts:1.18.2 - Maven Central"
[13]: https://www.baeldung.com/hibernate-spatial?utm_source=chatgpt.com "Introduction to Hibernate Spatial"
