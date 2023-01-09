---
layout: post
title: Hierarchical caches with Spring
tags: java spring
---

You have a large database containing a lot of data that you don't want to stress too often. This data is added to the database gradually over time. You want to provide read-only access to this data for your end users, while also ensuring that the integrity of the data is not compromised by a high volume of transactions. Additionally, you want to perform calculations on this data, such as descriptive statistics and differentials.

To begin with, you create a simple class called Database that loads a time series for you.

```java
public class Database {
    public double[] load(String series) {
        ... // horribly expensive database access goes here
    }
}
```

We want to calculate the sum of the time series returned by the database, producing an integral:

```java
public class Integrator {

    private Database database;

    public double[] run(String series) {
        double[] data = database.load(series);
        double[] result = new double[data.length];
        double sum = 0;
        for (int i = 0; i < data.length; i++) {
            sum += data[i];
            result[i] = sum;
        }
        return result;
    }
}
```

There are significant problems with this design. If we add a single new data point to the time series in the database, we have to retrieve the entire time series again. Even if there is no new data, if we request the integrated time series, the original time series will still be loaded and the calculation will be performed.

One solution to this problem is to use memoization, but because this is a stateful system, our caching strategy cannot be overly simplistic. To address these issues, we will use a range of Spring caching tools.

We will begin by adding caching to the `Database` and `Integrator` services.

```java
public class Integrator {

    ...

    @Cacheable(cacheNames = "integral", key = "#series")
    public double[] run(String series) {
        ...
    }
}

public class Database {

    ...

    @Cacheable(cacheNames = "nominal", key = "#series")
    public double[] load(String series) {
        ...
    }
}
```

We have made progress by memoizing and caching everything. Once written, the cache remains indefinitely. The next step is crucial because we need to coordinate cache eviction with the arrival of new data. To achieve this, we will introduce a new abstraction called a `Repository`:

```java
public class Repository {

    ...

    private Database database;

    @Caching(
            evict = @CacheEvict(value = "integral", key = "#series"),
            put = @CachePut(value = "nominal", key = "#series")
    )
    public double[] update(String series, double value) {
        double[] e = database.load(series);
        double[] d = new double[e.length + 1];
        System.arraycopy(e, 0, d, 0, e.length);
        d[e.length] = value;
        return d;
    }

    @Caching(evict = {
            @CacheEvict(value = "nominal", key = "#secret"),
            @CacheEvict(value = "integral", key = "#secret")
    })
    public void reset(String series) {
    }
}
```

There are multiple rather subtle points about this design. First, the method
`update` doesn't update the database, but rather

- reads the value from either the "nominal" cache or the database by executing `database.load()`
- adds a value at the end of the loaded series
- stores the new extended series in the "nominal" cache: `@CachePut`
- drops the "integral" cache for the new series: `@CacheEvict`

There are a few key points to consider in this design. First, the update method does not update the database directly. Instead, it:

- reads the value from either the "nominal" cache or the database using `database.load()`
- adds a value to the end of the loaded series
- stores the new, extended series in the "nominal" cache using `@CachePut`
- drops the "integral" cache for the new series using `@CacheEvict`


Second, the reset method does not access the database at all. It simply resets both the "nominal" and "integral" caches using annotations.

Some may argue that this approach is unnecessarily complex and that they could implement the necessary caching rules themselves more quickly. However, this approach becomes more practical when there are not just one, but many hierarchical calculations that need to be run on the data. By specifying functional dependencies in the Java code and caching policies in the annotations, concerns are separated and the code becomes more reusable and testable.
