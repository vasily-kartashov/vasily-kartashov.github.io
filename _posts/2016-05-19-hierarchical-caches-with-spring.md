---
layout: post
title: Optimizing Database Performance with Hierarchical Caching in Spring
tags: java spring
---

In large-scale applications, optimizing database performance is crucial. This article presents a sophisticated approach to enhancing read-only access and calculation efficiency using Spring's caching mechanisms. We'll explore how to implement a hierarchical caching strategy that significantly reduces database load and improves overall system performance.

## The Challenge

Consider a system with two primary components:

1. A Database class that retrieves time series data:

```java
public class Database {
    public double[] load(String series) {
        ... // horribly expensive database access goes here
    }
}
```

2. An Integrator class that performs calculations on this data:

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

This basic implementation has several inefficiencies:

- It reloads the entire time series when new data is added.
- It recalculates the integral even when the underlying data hasn't changed.

## The Solution: Hierarchical Caching

To address these issues, we'll implement a multi-layered caching strategy using Spring's caching annotations.

### Step 1: Basic Caching

First, we'll add caching to both the Database and Integrator classes:

```java
public class Database {
    @Cacheable(cacheNames = "nominal", key = "#series")
    public double[] load(String series) {
        // Implementation
    }
}

public class Integrator {
    @Cacheable(cacheNames = "integral", key = "#series")
    public double[] run(String series) {
        // Implementation
    }
}
```

### Step 2: Coordinated Cache Management

To maintain cache consistency when new data arrives, we introduce a Repository class:

```java
public class Repository {
    private Database database;

    @Caching(
        evict = @CacheEvict(value = "integral", key = "#series"),
        put = @CachePut(value = "nominal", key = "#series")
    )
    public double[] update(String series, double value) {
        double[] existing = database.load(series);
        double[] updated = new double[existing.length + 1];
        System.arraycopy(existing, 0, updated, 0, existing.length);
        updated[existing.length] = value;
        return updated;
    }

    @Caching(evict = {
        @CacheEvict(value = "nominal", key = "#series"),
        @CacheEvict(value = "integral", key = "#series")
    })
    public void reset(String series) {
        // Cache reset logic
    }
}
```

## Key Design Considerations

### Update Mechanism

The update method in Repository manages cache updates efficiently:
- _Retrieves existing data_: It fetches the current time series from the "nominal" cache or database. Example: If the series `AAPL` contains `[100, 101, 102]`, it retrieves this array.
- _Appends a new value_: It creates a new array with the existing data plus the new value at the end. Example: If updating AAPL` with `103`, the new array becomes `[100, 101, 102, 103]`.
- _Updates the "nominal" cache_: It stores the new array in the cache, replacing the old one. Example: The "nominal" cache for `AAPL` now contains `[100, 101, 102, 103]`.
- _Invalidates the "integral" cache_: It removes the corresponding entry from the "integral" cache. Example: The cached integral for `AAPL` is deleted, forcing recalculation on next access.

### Cache Reset

The reset method provides a way to clear cached data:
- _Clears both caches_: It removes entries from both "nominal" and "integral" caches for a given series. Example: Calling `reset("AAPL")` removes `AAPL` data from both caches.
- _No database interaction_: It only affects the caches, not the underlying database. Example: After reset, the next data request will trigger a fresh database load.

### Separation of Concerns: This design separates different aspects of the system:

- _Functional dependencies in Java code_: The actual data processing logic is in the Java methods. Example: The integration calculation in the Integrator class.
- _Caching policies in annotations_: Cache behavior is defined using Spring annotations. Example: `@Cacheable(cacheNames = "integral", key = "#series")` on the `Integrator.run()` method.

## Conclusion

While this hierarchical caching approach may seem complex initially, it proves highly effective in systems with multiple interdependent calculations. By leveraging Spring's caching annotations, we can significantly reduce database load, improve response times, and create a more scalable architecture.

This strategy is particularly valuable in read-heavy systems where data updates are less frequent than read operations. It allows for efficient data access and computation while maintaining data consistency across the caching layers.