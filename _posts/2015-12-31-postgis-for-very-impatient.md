---
layout: post
title: PostGIS for very impatient
---

As you may know, PostGIS is a PotgreSQL extension for working with spatial data, such as points, lines nad polygons.8
Let's start with pulling a docker image with PostGIS already install, which almost always the most confortable option

    sudo apt-get install docker.io
    sudo docker run --name postgis -e POSTGRES_PASSWORD=postgis -d mdillon/postgis:9.4

Login to the `psql` console

    sudo docker exec -it postgis psql -U postgres

and in the prompt execute the following command to check the version of the PostgreSQL / PostGIS

```sql
SELECT version();
SELECT postgis_full_version();
```

Docker runs in the container so you'll need the IP address if you want to access PostgeSQL running in the container

    sudo docker inspect postgis | grep IPAddress

Now we have a connection string and can use `pgcli` to access the database, which is a much nicer user interface that `psql`.

    pgcli -h 172.17.0.2 -U postgres

Let's create a simple spring boot project to play around with queries. The source can be found on [GitHub](https://github.com/vasily-kartashov/postgis-spring-data-jpa-example)

Initliaze the application

    spring.jpa.database-platform = org.hibernate.spatial.dialect.postgis.PostgisDialect
    spring.jpa.show-sql = true
    spring.jpa.hibernate.ddl-auto = update

    spring.datasource.url = jdbc:postgresql://172.17.0.1:5432/postgres?user=postgres&password=postgis
    spring.datasource.driver-class-name = org.postgresql.Driver

The device entity pretty much just concerns with the location of the device

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

Important is to define the custom type for the location attribute. This way Hibernate links Java geometry types and the SQL geometry type (??? is it true ???)

The hibernate dialect exposes PostGIS functions to JPQL and lets us to send queries like following

```java
public interface DeviceRepository extends CrudRepository<Device, String> {

    @Query("SELECT d FROM Device AS d WHERE within(d.location, :polygon) = TRUE")
    List<Device> findWithinPolygon(@Param("polygon") Polygon polygon);
}
```

You can take a pick at the class `org.hibernate.spatial.dialect.postgis.PostgisDialect` to see what functions are available on JPQL side and how they map to PostGIS functions

... - todo list them all?

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

@todo extend with other functions
@todo proper GIS data types
@todo convertes to make it more domain specific
