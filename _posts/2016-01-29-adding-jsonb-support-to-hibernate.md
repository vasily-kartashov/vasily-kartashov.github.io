---
layout: post
title: Adding JSONB support to Hibernate
---

While `@Converter`s are convenient to translate JSON objects to and from domain objects, there is an important limitation to this approach.
It is the fact that you cannot use any of the JSON functionalities in your JPQL queries and so you resorted to use native SQL queries.

One way to solve this problem is to write your custom Hibernate dialect, that supports JSONB type and functions and exposes them to the JPQL level.
You can see the source code for the following note on [GitHub](https://github.com/vasily-kartashov/postgis-spring-data-jpa-example/commit/8e2409def78b611bcb3d18d070e36ab65c61443f)

We start with a short explanation of what we're trying to achieve. Firstly, we would love to be able to have custom types backed by JSON to be available in the JPA entities.
Secondly, we want to be able to use `jsonb_*` functions in our JPLQ queries. Thirdly, adding a new type mapped to a JSONB column should be at most few lines of code.

Let's start with a custom Hibernate dialect. I will base it on PostgisDialect, as I plan to use both PostGIS and JSON in my application. We need to override a function `registerTypesAndFunctions`.

```java
@Override
protected void registerTypesAndFunctions() {
    super.registerTypesAndFunctions();
    registerColumnType(JSONTypeDescriptor.INSTANCE.getSqlType(), "jsonb");
    registerFunction("extract",
            new StandardSQLFunction("jsonb_extract_path_text", StandardBasicTypes.STRING));
}						    
```

Registering column type means that we explain to Hibernate how to deal with the situation when the database says that the target column is of type `jsonb`.
The class `JSONTypeDescriptor` registers the name of the SQL type, and the convertes from and to database object, which are called `binder` and `extractor`.
The code is rather trivial and is based on the fact that JSONB arrives as a `PGobject` which is just a tagged string.

Registering function means that we want to be able to use a function called `jsonb_extract_path_text` in our code, and the fact that it has a standard form of
`f(x, y, ...)` we use class `StandardSQLFunction` to explain how to translate it into the plain SQL. Also rather trivial.

Having a JSON string is just half of the problem solved. We now need to translate between domain objects and JSON strings. We will use Jackson to do the heavy lifting for us

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

The intricacies of wrapping and unwrapping escape me, so I just based my design on other custom types from PostGIS dialect.

The next step is to chain the two converters together:

    Domain Object <-- JavaTypeDescriptor --> String <-- SqlTypeDescriptor --> JSONP

We create an abstract class that chains two descriptors together:

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

This is all the non-domain specific code that we need. We can now proceed with our entities and repositories. Let's define a `Device` entity

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

And an embedded `Status` object

```java
public class Status {

    private double stateOfCharge;

    private String lifeCycle;

    ...
}
```

And now let's define the `StatusType` that would bind the Java and SQL type descriptors

```java
public class StatusType extends JSONBackedType<Status> {

    private final static JSONBackedTypeDescriptor<Status> descriptor =
            new JSONBackedTypeDescriptor<>(Status.class);

    public StatusType() {
        super(descriptor);
    }
}
```

So now we can start using JSONB in our queries. For example we can easily find all the devices with state of charge greater than 10%

```java
public interface DeviceRepository extends CrudRepository<Device, String> {

    @Query("SELECT d FROM Device d WHERE CAST(extract(d.status, 'stateOfCharge') float) > 0.1")
    List<Device> findHealthyDevices();

    ...
}
```

The JPQL query will be tranlated in the following SQL query

```sql
select device0_.id as id1_0_,
       device0_.location as location2_0_,
       device0_.status as status3_0_
  from devices device0_
 where cast(jsonb_extract_path_text(device0_.status, 'stateOfCharge') as float4) > 0.1
```

Which is pretty much what I wanted in the first place.
