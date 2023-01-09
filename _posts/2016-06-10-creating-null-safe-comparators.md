---
layout: post
title: Creating null-safe comparators in Java
tags: java
---

Sorting domain objects can be challenging because it may require significant effort to consider all edge cases and produce a robust sorting order that meets business requirements. For example, consider a client database where users are sorted first by city, then by zip code, then by last name, and finally by first name. If any of these properties are not set, we want to move the user to the end of the list. It may also be necessary to ignore the case when comparing cities. An example of a simple result set in tabulated form is shown below:

    | Brisbane  | 4000 | Burke     | Jason   |
    | Brisbane  | null | Appleseed | Frank   |
    | melbourne | 3001 | Collins   | Lindon  |
    | Melbourne | 3003 | Collins   | Grant   |
    | null      | 1000 | null      | Matthew |

The corresponding User class could look like this:

```java
class User {
    @Nullable public String getCity() { ... }
    @Nullable public Integer getZipCode() { ... }
    @Nullable public String getFirstName() { ... }
    @Nullable public String getLastName() { ... }
}
```

Although it's not an overly complex task, composing multiple comparators can be generalized. Here is a pseudo-code algorithm for this purpose:

    result = 0
    for (comparator : comparators):
        result = comparator.compare(a, b)
        if (result != 0):
            break

Using the flexibility of Java 8, we can create a generic `ComparatorBuilder` to simplify this task, particularly in regards to null checks:

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

For example, we can create a comparator for the `User` class as follows:

```java
Comparator<User> comparator = new ComparatorBuilder<User>()
        .by(User::getCity, String::toLowerCase)
        .by(User::getZipCode)
        .by(User::getLastName)
        .by(User::getFirstName)
        .build();
Collections.sort(users, comparator);
```

Overall, this approach is quite neat compared to the alternatives.
