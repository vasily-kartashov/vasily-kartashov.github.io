---
layout: post
title: How to define the base case for a function that accepts collections
tags: java
---

Sometimes, it is necessary to check if all elements of a collection are the same. A trivial solution to this problem is to compare each element to the first element in the collection and return `true` if they are all the same. For simplicity, we can ignore `null` elements in this comparison:

```java
boolean same(List<?> elements) {
	Object current = elements[0];
	for (int i = 1; i < elements.length; i++) {
	    Object next = elements[i];
	    if (!current.equals(next)) {
	       return false;
	    }
	    current = next;
	}
	return true;
}
```

However, it may not be obvious what to do with an empty collection. The question of the base case arises: are all elements of an empty collection the same or not?

```java
...
if (elements.isEmpty()) {
   return ?;
}
...
```

One way to reason about it is to start with the above-mentioned method and consider how it should work for smaller collections.

> If the method returns `true` for a given collection, then removing an element from this collection should not change the result. 

For example, if `same(Lists.of(a, b, c))` returns `true`, then `same(Lists.of(a, b))` should also return `true`, and by continuation `same(Collections.emptyList())` should also return `true`. This is important because a logically sound API is easier to work with.

Such method contracts usually make a great addition to documentation and provide you with a multitude of scenarios to be covered with unit tests.
