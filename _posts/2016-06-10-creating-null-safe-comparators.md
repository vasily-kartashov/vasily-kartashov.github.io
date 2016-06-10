---
layout: post
title: Creating null-safe comparators in Java
---

Domain objects can be quite tricky to sort. Depending on business requirements it
my take considerable effort to look for all edge cases while producing a robust
sorting order.

Let's say we have client database where we sort users by their city first,
then by zip code, then by last name and then by first name. If any of those
properties are not set we want to move users to the end of the list.
You also may want to ignore case when comparing cities. Here's an
example of a simple result set in a tabulated form

    | Brisbane  | 4000 | Burke     | Jason   |
    | Brisbane  | null | Appleseed | Frank   |
    | melbourne | 3001 | Collins   | Lindon  |
    | Melbourne | 3003 | Collins   | Grant   |
    | null      | 1000 | null      | Matthew |

And the corresponding `User` class might look like following

```java
class User {
    @Nullable public String getCity() { ... }
    @Nullable public Integer getZipCode() { ... }
    @Nullable public String getFirstName() { ... }
    @Nullable public String getLastName() { ... }
}
```

Now it's not that's unbearably complex task or anything, or that we don't know
how to do that. It's the fact that composing multiple comparators together can
be easily generalized. Here's the pseudo-code of this wonderful algorithm

    for (comparator : comparators):
        result = comparator.compare(a, b)
        if (result != 0):
            return result
        return 0

Now given all the flexibility of Java 8 we can create a generic `ComparatorBuilder`
that would help us to simplify this daunting task, especially around `null` checks

```java
public class ComparatorBuilder<T> {

    private final List<Comparator<T>> steps = new ArrayList<>();

    public <S extends Comparable<S>> ComparatorBuilder<T> by(Function<T, S> property) {
        return by(property, Function.identity());
    }

    public <S extends Comparable<S>, Q> ComparatorBuilder<T> by(Function<T, Q> property,
                                                                Function<Q, S> converter) {
        steps.add((T a1, T a2) -> {
            Q q1 = property.apply(a1);
            Q q2 = property.apply(a2);
            S s1 = q1 == null ? null : converter.apply(q1);
            S s2 = q2 == null ? null : converter.apply(q2);
            if (s1 == null) {
                return (s2 == null) ? 0 : -1;
            } else {
                return (s2 == null) ? 1 : s1.compareTo(s2);
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

So in our case we can create comparator for our `User` class as follows

```java
Comparator<User> comparator = new ComparatorBuilder<>()
        .by(User::getCity, String::toLowerCase)
        .by(User::getZipCode)
        .by(User::getLastName)
        .by(User::getFirstName)
        .build();

Collections.sort(users, comparator);
```

I think it's kind of neat, considering the alternatives.
