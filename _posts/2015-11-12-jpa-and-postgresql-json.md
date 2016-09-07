---
layout: post
title: Using JPA with PostgreSQL JSON
---

From time to time database developers are facing the problem of storing schema-free data, be it user preferences or a dynamic set of telemetry readings. If the set of keys is unknown or changed all the time, then normalizing the data is not an option. Instead most developers 'transpose' the table and produce the following kind of schema

    | sensor_id | location        |
    +-----------+-----------------+
    |         1 |         unknown |
    ...

    | status_field | status_value | sensor_id |
    +--------------+--------------+-----------+
    | temperature  | "0.9"        |         1 |
    | healthy      | "false"      |         1 |
    ...

There are many issues with this design including unnecessary join, single type (usually as string or a blob), single layer of indirection and so on. We've all been there at one point in our careers. With the advance of NoSQL databases the attention shifted to more flexible data structures. The most radical argument of the proponents of such move essentially boiled down to highlighting this kind of problems as mentioned above.

A more sensible approach that was to store the status data in the same table as sensors and use some structured format such as XML or JSON, like following

    | sensor_id | sensor_location | sensor_status           |
    +-----------+-----------------+-------------------------+
    |         1 |         unknown | { "temperature" : 0.9,  |
    |           |                 |   "healthy" : false }   |
    ...

The great news about this solution is that if you do that in PostgreSQL and declare the type of the STATUS field to be `json` or even better `jsonb`, then the status data is stored not as a string but as a structured binary data, and you can use this structure in your SQL queries. For example it's now possible to quickly find all unhealthy sensors

```sql
SELECT sensor_id
  FROM sensors
 WHERE (sensor_status->>'healthy')::boolean = false;
```

Now we have structured data in the database, and SQL queries. Can we also keep this structure in the application logic? For that we can use converters that would do the heavy ugly lifting for us. Let's start by creating the Status class. We will be using Jackson for JSON handling, and these are the annotations that we need to help Jackson to see through.

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
