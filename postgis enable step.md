 **PostGIS installation step** on your PostgreSQL setup yet.
---

### ✅ How to Install PostGIS on Windows (PostgreSQL 13)

#### 🔹 1. **Use StackBuilder (Official Method)**

StackBuilder is bundled with PostgreSQL on Windows. Here's how to install PostGIS:

1. Open **StackBuilder** (search in Start menu).
2. Select your PostgreSQL 13 installation.
3. In the list of available packages, find **PostGIS** (e.g., “PostGIS 3 for PostgreSQL 13”).
4. Install it — this will place the required files (`postgis.control`, `.sql`, `.dll`) in the correct directories.

---

#### 🔹 2. **Verify Installation**

After installation, open `psql` or pgAdmin and run:

```sql
CREATE EXTENSION postgis;
```

Then confirm with:

```sql
SELECT PostGIS_Full_Version();
```

---

### 🧠 Bonus Tip for Your VTS Pipeline

Once PostGIS is installed, your Spring Boot app will be able to use spatial types like `Point`, `Polygon`, etc. Make sure your entity uses:

```java
@Type(type = "org.hibernate.spatial.GeometryType")
@Column(columnDefinition = "geometry(Point,4326)")
private Point location;
```

And your `application.yml` includes:

```yaml
spring:
  jpa:
    properties:
      hibernate:
        dialect: org.hibernate.spatial.dialect.postgis.PostgisDialect
```

---

Let me know once PostGIS is installed — I can help you  a spatial entity and write geofence queries that plug directly into your Kafka Streams pipeline.


your StackBuilder interface is showing **two PostGIS bundles** under “Spatial Extensions”:

- **PostGIS 3.4 Bundle for PostgreSQL 13**
- **PostGIS 3.5 Bundle for PostgreSQL 13**

You only need to install **one** of them — preferably the **latest**, which is **PostGIS 3.5 Bundle v3.5.3**. That will give you the most up-to-date spatial features, bug fixes, and compatibility with tools like Hibernate Spatial and JTS.

---

### ✅ What to Do Next

1. **Select only**:  
   `✔ PostGIS 3.5 Bundle for PostgreSQL 13 (64 bit) v3.5.3`

2. Click **Next** and complete the installation.

3. After installation, open **pgAdmin** or `psql` and run:

   ```sql
   CREATE EXTENSION postgis;
   ```

4. Verify it with:

   ```sql
   SELECT PostGIS_Full_Version();
   ```

---

### 🧠 After That: Ready for Spatial Entities

Once PostGIS is active, you can start using spatial types like `Point`, `Polygon`, and `Geometry` in your Spring Boot entities. I can help you scaffold a full entity with geofence logic, spatial indexing, and Kafka-triggered alerts if you’re ready to plug it into your VTS pipeline.
