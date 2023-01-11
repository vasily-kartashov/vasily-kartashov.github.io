---
layout: post
title: Adding JSONB support to Hibernate
tags: hibernate postgresql
---

As you may be aware, the use of `@Converters` allows for the seamless translation of JSON objects to and from domain objects. However, there is a limitation to this approach in that it prohibits the use of JSON functionalities in JPQL queries, forcing one to rely solely on native SQL queries. To overcome this limitation, a custom Hibernate dialect that supports JSONB type and functions can be implemented, thereby exposing them to the JPQL level.

To accomplish this, we can create a custom Hibernate dialect that is based on `PostgisDialect`, as we plan to incorporate both PostGIS and JSON in our application. This can be achieved by overriding the `registerTypesAndFunctions` function. The source code for this implementation can be found on [GitHub](https://github.com/vasily-kartashov/postgis-spring-data-jpa-example/commit/8e2409def78b611bcb3d18d070e36ab65c61443f).

Our objective is to achieve three key goals:

* the ability to have custom types backed by JSON to be available in JPA entities
* the ability to utilize `jsonb_*` functions in JPLQ queries
* the ability to easily add new types mapped to JSONB columns with minimal coding.

By creating a custom Hibernate dialect, we can accomplish these goals, making our development process more efficient and streamlined.

Let's start by creating a custom Hibernate dialect, basing it on `PostgisDialect`, as we plan to use both PostGIS and JSON in our application. We need to override the `registerTypesAndFunctions` function:

```java
@Override
protected void registerTypesAndFunctions() {
    super.registerTypesAndFunctions();
    registerColumnType(JSONTypeDescriptor.INSTANCE.getSqlType(), "jsonb");
    registerFunction("extract",
            new StandardSQLFunction("jsonb_extract_path_text", StandardBasicTypes.STRING));
}
```

When registering a column type, we inform Hibernate on how to handle instances where the target column is of type `jsonb`. The `JSONTypeDescriptor` class registers the SQL type name and the converters that translate between the database object and the domain object, which are known as "binder" and "extractor". The process is relatively simple, as JSONB is received as a `PGobject` which is simply a tagged string.

By registering a function, we are able to use that function, such as `jsonb_extract_path_text`, in our code. The `StandardSQLFunction` class is utilized to explain how to translate the standard form of `f(x, y, ...)` into plain SQL. This process is also relatively straightforward.

However, just having a JSON string is only half of the solution. We also need to translate between domain objects and JSON strings. To accomplish this, we use the Jackson library for its powerful data binding capabilities:

```java
public class JSONBackedTypeDescriptor<T> extends AbstractTypeDescriptor<T> {

    private static final Logger logger = LoggerFactory.getLogger(JSONBackedTypeDescriptor.class);
    private static final ObjectMapper objectMapper = new ObjectMapper();

    public JSONBackedTypeDescriptor(Class<T> type) {
        super(type);
        JavaTypeDescriptorRegistry.INSTANCE.addDescriptor(this);
    }

    @Override
    public String toString(T value) {
        try {
            return objectMapper.writeValueAsString(value);
        } catch (JsonProcessingException e) {
            logger.warn("Cannot convert map {} to string", e);
            return "{}";
        }
    }

    @Override
    public T fromString(String string) {
        try {
            return objectMapper.readValue(string, getJavaTypeClass());
        } catch (IOException e) {
            logger.warn("Cannot read value from {}", string, e);
            return null;
        }
    }

    @Override
    public <X> X unwrap(T value, Class<X> type, WrapperOptions options) {
        if (value == null) {
            return null;
        }
        if (type.isAssignableFrom(value.getClass())) {
            return type.cast(value);
        }
        if (String.class.isAssignableFrom(type)) {
            return type.cast(toString(value));
        }
        throw unknownUnwrap(type);
    }

    @Override
    public <X> T wrap(X value, WrapperOptions options) {
        if (value == null) {
            return null;
        }
        if (value.getClass().isAssignableFrom(getJavaTypeClass())) {
            return getJavaTypeClass().cast(value);
        }
        if (value instanceof String) {
            return fromString((String) value);
        }
        throw unknownWrap(value.getClass());
    }
}
```

To ensure seamless integration, it is essential to link our two converters together. 

    Domain Object <-- JavaTypeDescriptor --> String <-- SqlTypeDescriptor --> JSONP

This includes creating a connection between the domain object, the `JavaTypeDescriptor`, the string representation, the `SqlTypeDescriptor`, and finally, the JSONP format. By creating an abstract class that facilitates this connection, we can ensure a smooth integration process:

```java
public abstract class JSONBackedType<T> extends AbstractSingleColumnStandardBasicType<T> {

    public JSONBackedType(JSONBackedTypeDescriptor<T> javaTypeDescriptor) {
        super(JSONTypeDescriptor.INSTANCE, javaTypeDescriptor);
    }

    @Override
    public String[] getRegistrationKeys() {
        return new String[] { getJavaTypeDescriptor().getJavaTypeClass().getCanonicalName() };
    }

    @Override
    public String getName() {
        return getJavaTypeDescriptor().getJavaTypeClass().getName();
    }
}
```

With all the necessary non-domain specific code in place, we can now proceed to create our entities and repositories. Let's take a look at our `Device` entity, defined as follows:

```java
@Entity
@Table(name = "devices")
public class Device {

    @Id
    @Column(name = "id")
    private String id;

    @Column(name = "location", nullable = false, columnDefinition = "geometry(Point,4326)")
    private Point location;

    @Column(name = "status", nullable = false, columnDefinition = "jsonb")
    @Type(type = "com.kartashov.postgis.types.StatusType")
    private Status status = new Status();

    ...
}
```

The `Device` entity is accompanied by an embedded `Status` object, which is defined as follows:

```java
public class Status {

    private double stateOfCharge;

    private String lifeCycle;

    ...
}
```

To link the Java and SQL type descriptors, we have defined the `StatusType` class, which extends the `JSONBackedType` class:

```java
public class StatusType extends JSONBackedType<Status> {

    private final static JSONBackedTypeDescriptor<Status> descriptor =
            new JSONBackedTypeDescriptor<>(Status.class);

    public StatusType() {
        super(descriptor);
    }
}
```

With our custom Hibernate dialect in place, we can now seamlessly integrate JSONB functionality into our JPQL queries. For instance, we can easily retrieve all devices with a state of charge greater than 10% by utilizing the following query within our `DeviceRepository` interface:

```java
public interface DeviceRepository extends CrudRepository<Device, String> {

    @Query("SELECT d FROM Device d WHERE CAST(extract(d.status, 'stateOfCharge') float) > 0.1")
    List<Device> findHealthyDevices();

    ...
}
```

This JPQL query will be translated into the following SQL query:

```sql
select device0_.id as id1_0_,
       device0_.location as location2_0_,
       device0_.status as status3_0_
  from devices device0_
 where cast(jsonb_extract_path_text(device0_.status, 'stateOfCharge') as float4) > 0.1
```

This implementation effectively achieves our goal of integrating JSONB functionality into our JPQL queries, allowing for easy and efficient data retrieval from our devices.
