---
layout: post
title: How to define the base case for a function that accepts collections
---

Imagine you have a function that checks if all elements of a collection are equal. A trivial solution would look like following (I deliberately ignore null's)

{% highlight java %}
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
{% endhighlight %}

It's all good except we expect the list to be non-empty. Easily enough we can add a simple check to the top of the method and return early.

{% highlight java %}
...
if (elements.isEmpty()) {
   return ?;
}
...
{% endhighlight %}

And here the question arises: what is the base case? Should I return `true` or `false`?

Start with the implied contract of the abovementioned method, i.e. if the method returns true for a given collection, then removing an element from this collection should not change the result.
Indeed, if `same(Lists.of(a, b, c))` returns true, then `same(Lists.of(a, b))` should also return true, and by continuation `same(Collections.emptyList())` should also return true. Why is that important?
Because a logically sound API is easier to work with.

Method contracts usually make a great addition to documentation and provide you with a multutude of scenarions to be covered with unit tests.
