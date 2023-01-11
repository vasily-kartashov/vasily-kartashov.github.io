---
layout: post
title: Hierarchical caches with Spring
tags: java spring
---

This article provides a solution to optimize the performance of a large database by using memoization and caching techniques in the `Database` and `Integrator` services. The goal is to reduce the stress on the database by providing read-only access to the data and perform calculations on this data. In this solution, you will use a range of Spring caching tools to achieve these goals.

To start, you create a simple class called `Database` that loads a time series for you:

```java
public class Database {
    public double[] load(String series) {
        ... // horribly expensive database access goes here
    }
}
```

You also create a class `Integrator` that uses this database to calculate the sum of the time series, producing an integral:

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

However, this design has some drawbacks; if a new data point is added to the time series in the database, the entire time series must be retrieved again. Even if there is no new data, if we request the integrated time series, the original time series will still be loaded and the calculation will be performed.

One solution to this problem is to use memoization, but because this is a stateful system, our caching strategy cannot be overly simplistic. To address these issues, we will use a range of Spring caching tools.

We will begin by adding caching to the `Database` and `Integrator` services:

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

We have made progress by memoizing and caching everything, but next step is crucial. We need to coordinate cache eviction with the arrival of new data. To achieve this, we will introduce a new abstraction called a `Repository`:

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

When it comes to this design, there are a few subtle details to keep in mind. The `update` method, for example, doesn't update the database directly. Instead, it:

- Retrieves the value from the "nominal" cache or the database by executing `database.load()`
- Adds a value to the end of the loaded series
- Stores the updated series in the "nominal" cache using the `@CachePut` annotation
- Clears the "integral" cache for the updated series using the `@CacheEvict` annotation

Additionally, the `reset` method does not access the database at all. It simply resets both the "nominal" and "integral" caches using annotations.

It's worth noting that this approach may seem complex at first glance, but it becomes more practical when there are multiple hierarchical calculations that need to be run on the data. By specifying functional dependencies in the Java code and caching policies in the annotations, the concerns are separated and the code becomes more reusable and testable.
