---
layout: post
title: Implementing Null-Safe Comparators in Java
tags: java
---

## Introduction

Sorting complex domain objects in Java can present challenges, particularly when considering multiple criteria and null values. This article explores an efficient approach to creating robust, null-safe comparators using Java 8 features.

## Problem Statement

Consider a scenario where users in a client database need to be sorted based on multiple criteria:

- City (case-insensitive)
- Zip code
- Last name
- First name

Additionally, null values for any of these properties should be handled by moving the corresponding user to the end of the list.

## Example Dataset

To illustrate the sorting requirements, consider the following sample dataset:

    | Brisbane  | 4000 | Burke     | Jason   |
    | Brisbane  | null | Appleseed | Frank   |
    | melbourne | 3001 | Collins   | Lindon  |
    | Melbourne | 3003 | Collins   | Grant   |
    | null      | 1000 | null      | Matthew |

This dataset demonstrates various scenarios, including case differences in city names, missing zip codes, and null values for both city and last name.

## Sample Data Structure

The User class might be structured as follows:

```java
class User {
    @Nullable public String getCity() { ... }
    @Nullable public Integer getZipCode() { ... }
    @Nullable public String getFirstName() { ... }
    @Nullable public String getLastName() { ... }
}
```

## Generalizing Comparator Composition

The process of composing multiple comparators can be generalized using the following algorithm:

    result = 0
    for (comparator : comparators):
        result = comparator.compare(a, b)
        if (result != 0):
            break

## Implementing a Generic ComparatorBuilder

Leveraging Java 8's functional interfaces, we can create a `ComparatorBuilder` class to simplify the creation of complex, `null`-safe comparators:

```java
public class ComparatorBuilder<T> {

    private final List<Comparator<T>> steps = new ArrayList<>();

    public <S extends Comparable<S>>
    ComparatorBuilder<T> by(Function<T, S> property) {
        return by(property, Function.identity());
    }

    public <S extends Comparable<S>, Q>
    ComparatorBuilder<T> by(Function<T, Q> property, Function<Q, S> converter) {
        steps.add((T a1, T a2) -> {
            Q q1 = property.apply(a1);
            Q q2 = property.apply(a2);
            S s1 = q1 == null ? null : converter.apply(q1);
            S s2 = q2 == null ? null : converter.apply(q2);
            if (s1 == null) {
                return (s2 == null) ?  0 : 1;
            } else {
                return (s2 == null) ? -1 : s1.compareTo(s2);
            }
        });
        return this;
    }

    public Comparator<T> build() {
        return (T a, T b) -> {
            int result = 0;
            for (Comparator<T> step : steps) {
                result = step.compare(a, b);
                if (result != 0) {
                    break;
                }
            }
            return result;
        };
    }
}
```

## Usage Example

To create a comparator for the `User` class using the `ComparatorBuilder`:

```java
Comparator<User> comparator = new ComparatorBuilder<User>()
        .by(User::getCity, String::toLowerCase)
        .by(User::getZipCode)
        .by(User::getLastName)
        .by(User::getFirstName)
        .build();
Collections.sort(users, comparator);
```

## Conclusion

This approach provides a clean, flexible, and type-safe method for creating complex comparators in Java. It effectively handles `null` values and allows for easy composition of multiple sorting criteria. By utilizing Java 8 features, we can create more maintainable and readable code for sorting operations.
