---
layout: post
title: Hierarchical caches with Spring
---

Imagine a huge database with a lot of data, that you don't want to stress too
often. The data is added to this database incrementally and over time you've
acquired quite a bit of it. You want to expose this data in a read-only manner
to your end users and you want to make sure that the liveness of the data is not
compromised by the volume of your transactions. On top of that you want to run
calculations based on this data, like descriptive statistics, differentials
and similar.

So the first thing first you create a simple class, let's call it `Database`,
that loads a time series for you

```java
public class Database {
    public double[] load(String series) {
        ... // horribly expensive database access goes here
    }
}
```

Over the time series returned by the database we would like to run a sum, thus
producing an integral like following

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

Now there are huge problems with this design. Imagine we add one more data
point to the time series in the database. We have to refetch the whole
time series. Worse, even if there's no new data available, if we ask for
integrated time series, the original time series will still be loaded and the
calculation performed.

Memoization, you say? This is a stateful system, so our caching strategy cannot
be too naive. To solve this problem we will use almost the whole arsenal of
Spring caching.

We'll start by adding caching to the `Database` and `Integrator`
services.

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

So far so good. Everything is memoized and caches, once written, stay so forever.
The next step is crucial. There's a coordination effort required
between cache eviction and new data arrival. We therefore introduce a new
abstraction that I will unapologetically call a `Repository`

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

Second, the method `reset` doesn't touch the database, in fact it doesn't do anything,
and only resets both caches - "nominal" and "integral" - through annotations.

Some might say, this is just horrible, I could write it all myself in no time,
by implementing the caching rules explicitly. And I am fairly sure it's true.
The approach described above will only start to make sense if there's not
a single integral function that you want to run over your data, but rather
a whole mesh of hierarchical calculations. Specifying the functional
dependencies in your java code and caching policy in the annotations keeps
concerns separated and allows better reuse and testability.
