---
layout: post
title: PostGIS for very impatient
tags: postgresql postgis
---

PostGIS is a powerful PostgreSQL extension designed to work with spatial data, including points, lines, and polygons. To begin experimenting with this extension, it is often easiest to pull a Docker image with PostGIS already installed. The following command can be used to do so:



    sudo apt-get install docker.io
    sudo docker run --name postgis -e POSTGRES_PASSWORD=postgis -d mdillon/postgis:9.4

Once the image is pulled, you can access the `psql` console by executing:

    sudo docker exec -it postgis psql -U postgres

At the prompt, you can check the version of PostgreSQL/PostGIS by running the following command:

```sql
SELECT version();
SELECT postgis_full_version();
```

As Docker runs the database in a container, you will need the IP address if you wish to access PostgreSQL from outside the container. The following command can be used to retrieve the IP address:

    sudo docker inspect postgis | grep IPAddress

ou can also use `pgcli`, a user-friendly cli tool, to access the database:

    pgcli -h 172.17.0.2 -U postgres

To further explore the capabilities of PostGIS, we can create a simple Spring Boot project. The source code for this project can be found on [GitHub](https://github.com/vasily-kartashov/postgis-spring-data-jpa-example)

Initliaze the application

    spring.jpa.database-platform = org.hibernate.spatial.dialect.postgis.PostgisDialect
    spring.jpa.show-sql = true
    spring.jpa.hibernate.ddl-auto = update

    spring.datasource.url = jdbc:postgresql://172.17.0.1:5432/postgres?user=postgres&password=postgis
    spring.datasource.driver-class-name = org.postgresql.Driver

In this example, we will use a device entity that concerns itself primarily with the location of the device:

```java
@Entity
@Table(name = "devices")
public class Device {

    @Id
    @Column(name = "id")
    private String id;

    @Column(name = "location", nullable = false, columnDefinition = "geometry(Point,4326)")
    private Point location;

    ...
}
```

It is essential to define a custom type for the location attribute, so that Hibernate can link Java geometry types with SQL geometry types.

The Hibernate dialect allows us to send queries like the following through JPQL:

```java
public interface DeviceRepository extends CrudRepository<Device, String> {

    @Query("SELECT d FROM Device AS d WHERE within(d.location, :polygon) = TRUE")
    List<Device> findWithinPolygon(@Param("polygon") Polygon polygon);
}
```

You can take a pick at the class `org.hibernate.spatial.dialect.postgis.PostgisDialect` to see what functions are available on JPQL side and how they map to PostGIS functions

Now we can test the search functionality

```java
Device device = new Device();
device.setId("de-001");
Point point = (Point) new WKTReader().read("POINT(5 5)");
device.setLocation(point);
deviceRepository.save(device);

Polygon polygon = (Polygon) new WKTReader().read("POLYGON((0 0,0 10,10 10,10 0,0 0))");
List<Device> devices = deviceRepository.findWithinPolygon(polygon);

System.out.println(devices);
```
