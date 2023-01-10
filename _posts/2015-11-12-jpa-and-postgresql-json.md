---
layout: post
title: Using JPA with PostgreSQL JSON
tags: java postgresql jpa
---

As database developers, we sometimes come across the challenge of storing data that doesn't have a fixed schema, such as user preferences or dynamic telemetry readings. When the set of keys is constantly changing or unknown, it's not possible to normalize the data. This is where "transposing" tables comes in - it's a common solution where a table is created with the following kind of schema:

    | sensor_id | location        |
    +-----------+-----------------+
    |         1 |         unknown |
    ...

    | status_field | status_value | sensor_id |
    +--------------+--------------+-----------+
    | temperature  | "0.9"        |         1 |
    | healthy      | "false"      |         1 |
    ...

However, this approach has its downsides - it can lead to unnecessary joins, single type (usually a string or a blob), and a single layer of indirection. With the rise of NoSQL databases, the focus has shifted towards more flexible data structures as a solution to these issues.

A more sensible approach is to store the status data in the same table as the sensors, using a structured format such as XML or JSON. For example, in PostgreSQL:

    | sensor_id | sensor_location | sensor_status           |
    +-----------+-----------------+-------------------------+
    |         1 |         unknown | { "temperature" : 0.9,  |
    |           |                 |   "healthy" : false }   |
    ...

This approach allows for structured data in the database and makes it possible to use SQL queries to quickly find all unhealthy sensors, for example:

```sql
SELECT sensor_id
  FROM sensors
 WHERE (sensor_status->>'healthy')::boolean = false;
```

It is also possible to maintain this structure in the application logic by using converters. The last step is to add a converter to your entity class to explain to your JPA implementation how to convert between the database data and your domain model, and back.

In this example, we're using Jackson for JSON handling and the annotations that we need to help Jackson to see through. Our converter contains a bit of boilerplate but it's worth it to have a structured data in the database and SQL queries.

```java
public class Status {

   private final Map<String, String> properties = new HashMap<>();

   @JsonAnyGetter
   public Map<Srting, Object> properties() {
       return properties;
   }

   @JsonAnySetter
   public void set(String key, Object value) {
       properties.put(key, value);
   }

   public Object get(String key) {
       if (properties.containsKey(key)) {
           return properties.get(key);
       } else {
           return null;
       }
   }
}
```

The converter that would convert between database data and the domain Status object contains quite a bit of boilerplate.

```java
@Converter
public class ConfigConverter implements AttributeConverter<Config, String> {

   private final ObjectMapper mapper = new ObjectMapper();

   @Override
   public String convertToDatabaseColumn(Status attribute) {
      try {
          return mapper.writeValueAsString(attribute);
      } catch (JsonProcessingException e) {
          throw new RuntimeException("Cannot serialize", e);
      }
   }

   @Override
   public Status convertToEntityAttribute(String dbData) {
      if (dbData == null) {
          return null;
      }
      try {
          return mapper.readValue(dbData, Status.class);
      } catch (IOException e) {
          throw new RuntimeException("Cannot deserialize", e);
      }
   }
}
```

The last thing you should do is to add a converter to your entity class to explain your JPA implementation how to convert between the database data and your domain model and back

```java
@Entity
@Table(name = "sensors")
public class Sensor {

    @Id
    @GeneratedValue
    @Column(name = "sensor_id")
    private Long id;

    @Column(name = "sensor_location")
    private String location;

    @Column(name = "sensor_status")
    @Convert(converter = StatusConverter.class)
    private Status status;

    ...
}
```

One tiny note. You will need to use `pgjdbc-ng` driver for this. At least this is the only one that worked for me.
