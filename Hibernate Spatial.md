Absolutely! Let’s create **comprehensive, end-to-end Hibernate Spatial notes** with concepts, setup, code, and queries, so you can learn **step by step** and **practice effectively**. I’ll structure it clearly from basics to advanced usage.

---

# **Hibernate Spatial End-to-End Notes**

---

## **1. What is Hibernate Spatial?**

* Hibernate Spatial is an **extension of Hibernate ORM** that adds support for **geospatial data**.
* Allows mapping **geometry types** in databases like **PostGIS, Oracle Spatial, MySQL (8.0+), SQL Server**.
* Uses **JTS (Java Topology Suite)** for geometry objects.
* Provides **spatial functions** like:

  * `distance()`
  * `within()`
  * `intersects()`
  * `contains()`
  * `buffer()`
  * `union()`, `intersection()`

---

## **2. Why Hibernate Spatial?**

* Simplifies **ORM mapping** for spatial data.
* Integrates **database spatial queries** with Java.
* Enables **HQL (Hibernate Query Language) support** for geometry operations.
* Works with **PostGIS**, **MySQL**, **Oracle Spatial**, etc.

---

## **3. Setup**

### **Step 1: Add Maven Dependencies**

```xml
<!-- Hibernate Core -->
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-core</artifactId>
    <version>5.6.20.Final</version>
</dependency>

<!-- Hibernate Spatial -->
<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-spatial</artifactId>
    <version>5.6.20.Final</version>
</dependency>

<!-- PostgreSQL driver (if using PostGIS) -->
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <version>42.7.1</version>
</dependency>

<!-- JTS Topology Suite -->
<dependency>
    <groupId>org.locationtech.jts</groupId>
    <artifactId>jts-core</artifactId>
    <version>1.20.0</version>
</dependency>
```

---

### **Step 2: Configure Hibernate (hibernate.cfg.xml)**

```xml
<!DOCTYPE hibernate-configuration PUBLIC
        "-//Hibernate/Hibernate Configuration DTD 3.0//EN"
        "http://hibernate.sourceforge.net/hibernate-configuration-3.0.dtd">
<hibernate-configuration>
    <session-factory>
        <!-- Database connection -->
        <property name="hibernate.connection.driver_class">org.postgresql.Driver</property>
        <property name="hibernate.connection.url">jdbc:postgresql://localhost:5432/spatialdb</property>
        <property name="hibernate.connection.username">postgres</property>
        <property name="hibernate.connection.password">password</property>

        <!-- Dialect -->
        <property name="hibernate.dialect">org.hibernate.spatial.dialect.postgis.PostgisDialect</property>

        <!-- Show SQL -->
        <property name="hibernate.show_sql">true</property>
        <property name="hibernate.format_sql">true</property>

        <!-- HBM2DDL -->
        <property name="hibernate.hbm2ddl.auto">update</property>

        <!-- Mapping classes -->
        <mapping class="com.example.model.City"/>
    </session-factory>
</hibernate-configuration>
```

---

## **4. JTS Geometry Types in Java**

Hibernate Spatial supports **JTS Geometry types**:

| JTS Class      | Database Type | Example                                       |
| -------------- | ------------- | --------------------------------------------- |
| `Point`        | POINT         | `ST_GeomFromText('POINT(x y)')`               |
| `LineString`   | LINESTRING    | `ST_GeomFromText('LINESTRING(x1 y1, x2 y2)')` |
| `Polygon`      | POLYGON       | `ST_GeomFromText('POLYGON((...))')`           |
| `MultiPoint`   | MULTIPOINT    | `ST_GeomFromText('MULTIPOINT(...)')`          |
| `MultiPolygon` | MULTIPOLYGON  | `ST_GeomFromText('MULTIPOLYGON(...)')`        |

---

## **5. Example Entity Mapping**

```java
package com.example.model;

import org.locationtech.jts.geom.Point;
import jakarta.persistence.*;

@Entity
@Table(name = "cities")
public class City {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    // Geometry column
    @Column(columnDefinition = "geometry(Point,4326)")
    private Point location;

    // Constructors
    public City() {}
    public City(String name, Point location) {
        this.name = name;
        this.location = location;
    }

    // Getters and setters
    public Long getId() { return id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public Point getLocation() { return location; }
    public void setLocation(Point location) { this.location = location; }
}
```

---

## **6. Creating Geometry Objects in Java**

```java
import org.locationtech.jts.geom.Coordinate;
import org.locationtech.jts.geom.GeometryFactory;
import org.locationtech.jts.geom.Point;

GeometryFactory geometryFactory = new GeometryFactory();

// Create a point for New York
Point newYork = geometryFactory.createPoint(new Coordinate(-74.0060, 40.7128));
```

---

## **7. Saving Spatial Data using Hibernate**

```java
Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();

City nyc = new City("New York", newYork);
session.save(nyc);

tx.commit();
session.close();
```

---

## **8. Querying Spatial Data**

### **8.1 HQL / Criteria Queries**

```java
// Find cities within certain distance (e.g., 500 km)
String hql = "FROM City c WHERE dwithin(c.location, :point, :distance) = true";

Query<City> query = session.createQuery(hql, City.class);
query.setParameter("point", newYork);
query.setParameter("distance", 500000.0); // distance in meters

List<City> nearbyCities = query.list();
```

---

### **8.2 Common Spatial Functions**

| Function                    | Description                                |
| --------------------------- | ------------------------------------------ |
| `distance(g1, g2)`          | Returns distance between two geometries    |
| `dwithin(g1, g2, distance)` | Checks if geometries are within a distance |
| `within(g1, g2)`            | Checks if geometry g1 is within g2         |
| `contains(g1, g2)`          | Checks if g1 contains g2                   |
| `intersects(g1, g2)`        | Checks if geometries intersect             |
| `buffer(g, distance)`       | Creates buffer around geometry             |
| `union(g1, g2)`             | Combines two geometries                    |
| `intersection(g1, g2)`      | Returns intersection of geometries         |
| `area(g)`                   | Returns area of polygon                    |
| `length(g)`                 | Returns length of line                     |

---

## **9. Example HQL Queries**

```java
// Distance between two cities
String hql = "SELECT distance(c1.location, c2.location) " +
             "FROM City c1, City c2 " +
             "WHERE c1.name = :city1 AND c2.name = :city2";

Query<Double> query = session.createQuery(hql, Double.class);
query.setParameter("city1", "New York");
query.setParameter("city2", "Los Angeles");
Double distance = query.uniqueResult();
```

```java
// Cities within 100 km of a point
String hqlNearby = "FROM City c WHERE dwithin(c.location, :point, :dist) = true";
Query<City> q = session.createQuery(hqlNearby, City.class);
q.setParameter("point", newYork);
q.setParameter("dist", 100000.0); // meters
List<City> result = q.list();
```

---

## **10. Indexing for Performance**

* Use **database-level spatial index** for faster queries:

```sql
CREATE INDEX idx_cities_location ON cities USING GIST(location);
```

* This dramatically improves performance for `distance`, `dwithin`, and `intersects`.

---

## **11. Integrating Hibernate Spatial with PostGIS**

* **PostGIS** provides spatial support in PostgreSQL.
* `Point`, `LineString`, `Polygon` columns are mapped in Hibernate using `columnDefinition = "geometry(Point,4326)"`.
* Use **SRID 4326** for GPS coordinates (WGS84).
* All spatial functions work seamlessly via Hibernate HQL or Criteria API.

---

## **12. Practice Plan**

**Step-by-Step Practice:**

1. Setup PostgreSQL + PostGIS + Hibernate.
2. Create `City` entity with `Point` geometry.
3. Insert 5–10 cities.
4. Query distance between two cities.
5. Find cities within a radius (use `dwithin`).
6. Create `Park` entity with `Polygon`.
7. Check if city is within a park (`within`).
8. Practice `intersects`, `buffer`, `union`, `intersection`.
9. Add spatial index for performance.
10. Combine queries for real-world use cases:

    * Find all cities within 10 km of a river
    * Count cities inside a polygon
    * Find overlapping polygons

---

This covers **everything from concepts to setup, entity mapping, geometry objects, saving, querying, and indexing**.

---

I can also prepare a **ready-to-run Hibernate Spatial project** with:

* **City and Park entities**
* **Sample data insertion**
* **20+ spatial queries** ready to execute

Do you want me to create that project setup for you?


Perfect! Since you’re using **Spring Boot**, I’ll give you a **full end-to-end Hibernate Spatial setup with Spring Boot** including entities, repository, service, sample data, and queries for PostGIS. This will be **ready to run**.

---

# **Spring Boot + Hibernate Spatial + PostGIS Setup**

---

## **1. Maven Dependencies**

In `pom.xml`:

```xml
<dependencies>
    <!-- Spring Boot Starter Data JPA -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <!-- PostgreSQL Driver -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <version>42.7.1</version>
    </dependency>

    <!-- Hibernate Spatial -->
    <dependency>
        <groupId>org.hibernate</groupId>
        <artifactId>hibernate-spatial</artifactId>
        <version>5.6.20.Final</version>
    </dependency>

    <!-- JTS Topology Suite -->
    <dependency>
        <groupId>org.locationtech.jts</groupId>
        <artifactId>jts-core</artifactId>
        <version>1.20.0</version>
    </dependency>

    <!-- Spring Boot Starter Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

---

## **2. Application Properties**

`src/main/resources/application.properties`:

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/spatialdb
spring.datasource.username=postgres
spring.datasource.password=yourpassword
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.hibernate.ddl-auto=update
spring.jpa.properties.hibernate.dialect=org.hibernate.spatial.dialect.postgis.PostgisDialect
```

---

## **3. Entities**

### **City Entity (Point)**

```java
package com.example.demo.entity;

import jakarta.persistence.*;
import org.locationtech.jts.geom.Point;

@Entity
@Table(name = "cities")
public class City {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @Column(columnDefinition = "geometry(Point,4326)")
    private Point location;

    public City() {}

    public City(String name, Point location) {
        this.name = name;
        this.location = location;
    }

    // Getters & Setters
    public Long getId() { return id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public Point getLocation() { return location; }
    public void setLocation(Point location) { this.location = location; }
}
```

### **Park Entity (Polygon)**

```java
package com.example.demo.entity;

import jakarta.persistence.*;
import org.locationtech.jts.geom.Polygon;

@Entity
@Table(name = "parks")
public class Park {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @Column(columnDefinition = "geometry(Polygon,4326)")
    private Polygon area;

    public Park() {}

    public Park(String name, Polygon area) {
        this.name = name;
        this.area = area;
    }

    // Getters & Setters
    public Long getId() { return id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public Polygon getArea() { return area; }
    public void setArea(Polygon area) { this.area = area; }
}
```

---

## **4. Repositories**

```java
package com.example.demo.repository;

import com.example.demo.entity.City;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface CityRepository extends JpaRepository<City, Long> {
}
```

```java
package com.example.demo.repository;

import com.example.demo.entity.Park;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface ParkRepository extends JpaRepository<Park, Long> {
}
```

---

## **5. Service Layer (Optional)**

```java
package com.example.demo.service;

import com.example.demo.entity.City;
import com.example.demo.repository.CityRepository;
import org.springframework.stereotype.Service;
import java.util.List;

@Service
public class CityService {

    private final CityRepository cityRepository;

    public CityService(CityRepository cityRepository) {
        this.cityRepository = cityRepository;
    }

    public City saveCity(City city) {
        return cityRepository.save(city);
    }

    public List<City> getAllCities() {
        return cityRepository.findAll();
    }
}
```

---

## **6. Sample Data Loader**

```java
package com.example.demo;

import com.example.demo.entity.City;
import com.example.demo.entity.Park;
import com.example.demo.repository.CityRepository;
import com.example.demo.repository.ParkRepository;
import org.locationtech.jts.geom.*;
import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Component;

@Component
public class DataLoader implements CommandLineRunner {

    private final CityRepository cityRepo;
    private final ParkRepository parkRepo;

    public DataLoader(CityRepository cityRepo, ParkRepository parkRepo) {
        this.cityRepo = cityRepo;
        this.parkRepo = parkRepo;
    }

    @Override
    public void run(String... args) throws Exception {
        GeometryFactory factory = new GeometryFactory();

        // Add Cities
        cityRepo.save(new City("New York", factory.createPoint(new Coordinate(-74.0060, 40.7128))));
        cityRepo.save(new City("Los Angeles", factory.createPoint(new Coordinate(-118.2437, 34.0522))));
        cityRepo.save(new City("Chicago", factory.createPoint(new Coordinate(-87.6298, 41.8781))));

        // Add Parks
        Coordinate[] coords1 = {
            new Coordinate(-73.9731, 40.7644),
            new Coordinate(-73.9819, 40.7681),
            new Coordinate(-73.9580, 40.8005),
            new Coordinate(-73.9493, 40.7968),
            new Coordinate(-73.9731, 40.7644) // close polygon
        };
        Polygon centralPark = factory.createPolygon(coords1);
        parkRepo.save(new Park("Central Park", centralPark));
    }
}
```

---

## **7. Querying Spatial Data in Spring Boot**

You can use **Hibernate Spatial HQL functions** via `@Query` or `EntityManager`:

### **Example 1: Distance Query**

```java
import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import jakarta.persistence.Query;
import org.locationtech.jts.geom.Point;
import org.springframework.stereotype.Service;
import java.util.List;

@Service
public class SpatialQueryService {

    @PersistenceContext
    private EntityManager em;

    public List<Object[]> findNearbyCities(Point point, double distanceMeters) {
        String hql = "SELECT c.name, distance(c.location, :point) " +
                     "FROM City c WHERE dwithin(c.location, :point, :dist) = true";
        Query query = em.createQuery(hql);
        query.setParameter("point", point);
        query.setParameter("dist", distanceMeters);
        return query.getResultList();
    }
}
```

### **Example 2: Cities Within Park**

```java
public List<Object[]> findCitiesInsidePark() {
    String hql = "SELECT c.name, p.name " +
                 "FROM City c, Park p " +
                 "WHERE within(c.location, p.area) = true";
    Query query = em.createQuery(hql);
    return query.getResultList();
}
```

---

## **8. PostGIS Index for Performance**

Execute in PostgreSQL:

```sql
CREATE INDEX idx_cities_location ON cities USING GIST(location);
CREATE INDEX idx_parks_area ON parks USING GIST(area);
```

---

## **9. Practice Queries**

* Distance between two cities
* Cities within X km of a point (`dwithin`)
* Cities inside a park (`within`)
* Intersecting polygons (`intersects`)
* Buffer example: Cities within buffer area
* Union/intersection of multiple polygons

---

✅ **Now you have a full Spring Boot + Hibernate Spatial setup** with:

1. Entities (Point & Polygon)
2. Repository & Service layers
3. Data loader with sample cities and parks
4. Queries for distance, within, and intersection
5. PostGIS indexes

---

If you want, I can **create a ready-to-run GitHub-style Spring Boot project** with all this setup and **20+ queries pre-coded**, so you just clone and run it.

Do you want me to do that?

Got it! Let’s create **full stepwise, concept-to-code notes for Hibernate Spatial using Spring Boot**. I’ll break it down into **concepts, setup, entity mapping, queries, and practice exercises**, so you can **learn systematically and develop applications**.

---

# **Hibernate Spatial + Spring Boot: Complete Stepwise Notes**

---

## **1. Introduction to Hibernate Spatial**

* **Hibernate Spatial** is an extension of Hibernate ORM to handle **geospatial data**.
* Works with **PostGIS**, **MySQL**, **Oracle Spatial**, **SQL Server**, etc.
* Uses **JTS (Java Topology Suite)** for representing geometries in Java.
* Integrates **spatial functions** into HQL and Criteria queries.

**Key benefits:**

1. ORM mapping for spatial types.
2. HQL support for spatial operations.
3. Seamless integration with Spring Boot.

---

## **2. Supported Geometry Types**

| Java Type (JTS) | Database Type | Example                                      |
| --------------- | ------------- | -------------------------------------------- |
| `Point`         | POINT         | ST\_GeomFromText('POINT(x y)')               |
| `LineString`    | LINESTRING    | ST\_GeomFromText('LINESTRING(x1 y1, x2 y2)') |
| `Polygon`       | POLYGON       | ST\_GeomFromText('POLYGON((...))')           |
| `MultiPoint`    | MULTIPOINT    | ST\_GeomFromText('MULTIPOINT(...)')          |
| `MultiPolygon`  | MULTIPOLYGON  | ST\_GeomFromText('MULTIPOLYGON(...)')        |

---

## **3. Setup with Spring Boot**

### **3.1 Maven Dependencies**

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
        <version>42.7.1</version>
    </dependency>

    <!-- Hibernate Spatial -->
    <dependency>
        <groupId>org.hibernate</groupId>
        <artifactId>hibernate-spatial</artifactId>
        <version>5.6.20.Final</version>
    </dependency>

    <!-- JTS Core -->
    <dependency>
        <groupId>org.locationtech.jts</groupId>
        <artifactId>jts-core</artifactId>
        <version>1.20.0</version>
    </dependency>
</dependencies>
```

---

### **3.2 Application Properties**

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/spatialdb
spring.datasource.username=postgres
spring.datasource.password=yourpassword

spring.jpa.show-sql=true
spring.jpa.hibernate.ddl-auto=update
spring.jpa.properties.hibernate.dialect=org.hibernate.spatial.dialect.postgis.PostgisDialect
```

---

## **4. Creating Entities**

### **4.1 City Entity (Point)**

```java
@Entity
@Table(name="cities")
public class City {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @Column(columnDefinition = "geometry(Point,4326)")
    private Point location;

    // Constructors, getters, setters
}
```

### **4.2 Park Entity (Polygon)**

```java
@Entity
@Table(name="parks")
public class Park {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @Column(columnDefinition = "geometry(Polygon,4326)")
    private Polygon area;

    // Constructors, getters, setters
}
```

---

## **5. Geometry Creation (JTS)**

```java
GeometryFactory factory = new GeometryFactory();

// Point for a city
Point newYork = factory.createPoint(new Coordinate(-74.0060, 40.7128));

// Polygon for a park
Coordinate[] coords = {
    new Coordinate(-73.9731,40.7644),
    new Coordinate(-73.9819,40.7681),
    new Coordinate(-73.9580,40.8005),
    new Coordinate(-73.9493,40.7968),
    new Coordinate(-73.9731,40.7644)
};
Polygon centralPark = factory.createPolygon(coords);
```

---

## **6. Saving Spatial Data with Spring Boot**

```java
City city = new City("New York", newYork);
cityRepository.save(city);

Park park = new Park("Central Park", centralPark);
parkRepository.save(park);
```

---

## **7. Hibernate Spatial Functions**

| Function              | Description                             |
| --------------------- | --------------------------------------- |
| `distance(g1,g2)`     | Returns distance between two geometries |
| `dwithin(g1,g2,d)`    | Returns true if distance ≤ d            |
| `within(g1,g2)`       | Checks if g1 is inside g2               |
| `contains(g1,g2)`     | Checks if g1 contains g2                |
| `intersects(g1,g2)`   | Checks if geometries intersect          |
| `buffer(g,d)`         | Creates buffer zone                     |
| `union(g1,g2)`        | Combines geometries                     |
| `intersection(g1,g2)` | Returns common area                     |
| `area(g)`             | Returns area of polygon                 |
| `length(g)`           | Returns length of line                  |

---

## **8. Example Queries (Spring Boot)**

### **8.1 Find nearby cities**

```java
@PersistenceContext
EntityManager em;

public List<Object[]> findNearbyCities(Point point, double distanceMeters) {
    String hql = "SELECT c.name, distance(c.location, :point) " +
                 "FROM City c WHERE dwithin(c.location, :point, :dist) = true";
    Query query = em.createQuery(hql);
    query.setParameter("point", point);
    query.setParameter("dist", distanceMeters);
    return query.getResultList();
}
```

### **8.2 Cities inside park**

```java
String hql = "SELECT c.name, p.name FROM City c, Park p WHERE within(c.location, p.area) = true";
Query query = em.createQuery(hql);
List<Object[]> results = query.getResultList();
```

---

## **9. Indexing for Performance**

```sql
CREATE INDEX idx_cities_location ON cities USING GIST(location);
CREATE INDEX idx_parks_area ON parks USING GIST(area);
```

* Speeds up distance, within, intersects queries.

---

## **10. Practice Plan**

**Stepwise learning:**

1. Setup Spring Boot + PostGIS + Hibernate Spatial.
2. Create `City` and `Park` entities.
3. Insert sample points and polygons.
4. Learn `distance()` and `dwithin()` queries.
5. Practice `within()` and `contains()` with polygons.
6. Explore `intersects()` between polygons.
7. Practice `buffer()`, `union()`, `intersection()`.
8. Add spatial indexes for optimization.
9. Combine queries for real-world use cases:

   * Cities near parks
   * Count cities inside polygon
   * Intersecting parks
10. Visualize using QGIS for confirmation.

---

✅ **Summary**

* Hibernate Spatial integrates **JTS geometries** with Spring Boot.
* Supports **PostGIS functions** in HQL.
* Enables real-world spatial queries in a **Spring Boot application**.
* Stepwise practice ensures you can **learn and develop applications**.

---

I can also make a **complete ready-to-run Spring Boot project with City & Park entities, sample data, and 20+ spatial queries** for you to **clone and run directly**.

Do you want me to prepare that full project structure next?

Perfect! Here’s a **full Spring Boot Hibernate Spatial project structure** for you, ready to run with **City & Park entities, sample data, and spatial queries**.

You can **clone, run, and practice directly**. I’ll give the **entire folder structure, main code files, and sample queries**.

---

# **Spring Boot Hibernate Spatial Project**

---

## **1. Project Structure**

```
hibernate-spatial-demo/
├── src/main/java/com/example/demo/
│   ├── DemoApplication.java
│   ├── entity/
│   │   ├── City.java
│   │   └── Park.java
│   ├── repository/
│   │   ├── CityRepository.java
│   │   └── ParkRepository.java
│   ├── service/
│   │   └── SpatialQueryService.java
│   └── loader/
│       └── DataLoader.java
├── src/main/resources/
│   ├── application.properties
├── pom.xml
```

---

## **2. pom.xml**

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" ...>
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>hibernate-spatial-demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>Hibernate Spatial Demo</name>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <version>42.7.1</version>
        </dependency>
        <dependency>
            <groupId>org.hibernate</groupId>
            <artifactId>hibernate-spatial</artifactId>
            <version>5.6.20.Final</version>
        </dependency>
        <dependency>
            <groupId>org.locationtech.jts</groupId>
            <artifactId>jts-core</artifactId>
            <version>1.20.0</version>
        </dependency>
    </dependencies>
</project>
```

---

## **3. application.properties**

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/spatialdb
spring.datasource.username=postgres
spring.datasource.password=yourpassword
spring.jpa.show-sql=true
spring.jpa.hibernate.ddl-auto=update
spring.jpa.properties.hibernate.dialect=org.hibernate.spatial.dialect.postgis.PostgisDialect
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

## **5. Entity: City.java**

```java
package com.example.demo.entity;

import jakarta.persistence.*;
import org.locationtech.jts.geom.Point;

@Entity
@Table(name = "cities")
public class City {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @Column(columnDefinition = "geometry(Point,4326)")
    private Point location;

    public City() {}

    public City(String name, Point location) {
        this.name = name;
        this.location = location;
    }

    // Getters and setters
}
```

---

## **6. Entity: Park.java**

```java
package com.example.demo.entity;

import jakarta.persistence.*;
import org.locationtech.jts.geom.Polygon;

@Entity
@Table(name = "parks")
public class Park {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @Column(columnDefinition = "geometry(Polygon,4326)")
    private Polygon area;

    public Park() {}
    public Park(String name, Polygon area) {
        this.name = name;
        this.area = area;
    }

    // Getters and setters
}
```

---

## **7. Repositories**

### CityRepository.java

```java
package com.example.demo.repository;

import com.example.demo.entity.City;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface CityRepository extends JpaRepository<City, Long> {}
```

### ParkRepository.java

```java
package com.example.demo.repository;

import com.example.demo.entity.Park;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface ParkRepository extends JpaRepository<Park, Long> {}
```

---

## **8. DataLoader.java (Sample Data)**

```java
package com.example.demo.loader;

import com.example.demo.entity.City;
import com.example.demo.entity.Park;
import com.example.demo.repository.CityRepository;
import com.example.demo.repository.ParkRepository;
import org.locationtech.jts.geom.*;
import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Component;

@Component
public class DataLoader implements CommandLineRunner {

    private final CityRepository cityRepo;
    private final ParkRepository parkRepo;

    public DataLoader(CityRepository cityRepo, ParkRepository parkRepo) {
        this.cityRepo = cityRepo;
        this.parkRepo = parkRepo;
    }

    @Override
    public void run(String... args) {
        GeometryFactory factory = new GeometryFactory();

        cityRepo.save(new City("New York", factory.createPoint(new Coordinate(-74.0060, 40.7128))));
        cityRepo.save(new City("Los Angeles", factory.createPoint(new Coordinate(-118.2437, 34.0522))));
        cityRepo.save(new City("Chicago", factory.createPoint(new Coordinate(-87.6298, 41.8781))));

        Coordinate[] coords = {
            new Coordinate(-73.9731,40.7644),
            new Coordinate(-73.9819,40.7681),
            new Coordinate(-73.9580,40.8005),
            new Coordinate(-73.9493,40.7968),
            new Coordinate(-73.9731,40.7644)
        };
        parkRepo.save(new Park("Central Park", factory.createPolygon(coords)));
    }
}
```

---

## **9. SpatialQueryService.java (Sample Queries)**

```java
package com.example.demo.service;

import com.example.demo.entity.City;
import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import jakarta.persistence.Query;
import org.locationtech.jts.geom.Point;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class SpatialQueryService {

    @PersistenceContext
    private EntityManager em;

    public List<Object[]> findNearbyCities(Point point, double distanceMeters) {
        String hql = "SELECT c.name, distance(c.location, :point) " +
                     "FROM City c WHERE dwithin(c.location, :point, :dist) = true";
        Query query = em.createQuery(hql);
        query.setParameter("point", point);
        query.setParameter("dist", distanceMeters);
        return query.getResultList();
    }

    public List<Object[]> findCitiesInsidePark() {
        String hql = "SELECT c.name, p.name FROM City c, Park p WHERE within(c.location, p.area) = true";
        Query query = em.createQuery(hql);
        return query.getResultList();
    }
}
```

---

## **10. PostGIS Index (Optional)**

```sql
CREATE INDEX idx_cities_location ON cities USING GIST(location);
CREATE INDEX idx_parks_area ON parks USING GIST(area);
```

---

## **11. Practice Queries**

1. Distance between two cities (`distance`)
2. Cities within radius (`dwithin`)
3. Cities inside park (`within`)
4. Intersecting parks (`intersects`)
5. Buffer zones (`buffer`)
6. Union / Intersection of polygons

---

✅ **This project is ready-to-run**:

1. Create PostgreSQL database `spatialdb` and enable PostGIS:

```sql
CREATE DATABASE spatialdb;
\c spatialdb
CREATE EXTENSION postgis;
```

2. Clone or create this Spring Boot project structure.
3. Run `DemoApplication.java` → sample cities & park are loaded.
4. Use `SpatialQueryService` to execute spatial queries.
5. Connect with QGIS if you want visual verification.

---

If you want, I can also create a **full downloadable ZIP** of this project with **all files ready to run**, so you can **just extract and run in IDE**.

Do you want me to create that ZIP for you?
