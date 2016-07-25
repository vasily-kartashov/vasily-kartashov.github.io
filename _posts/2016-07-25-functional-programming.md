---
layout: post
title: Functional programming
---

In functional programming in order to reason about the meaning of a symbol (variable or function name) you only really need to know 2 things --- current scope and the name of the symbol. If you have a purely functional language with immutability both of these are "static" concepts, meaning you can see both -- the current scope and the name --- just by looking at the source code.

In procedural programming if you want to answer the question what's the value behind x you also need to know how you got there, the scope and name alone are not enough. And this is what I would see as the biggest challenge because this execution path is a "runtime" property and can depend on so many different things, that most people learn to just debug it and not to try and recover the execution path.

Needless to say all of the ideas above were ~~shamelessly stolen~~ reinterpreted from this [essay by Dijkstra](https://www.cs.utexas.edu/users/EWD/transcriptions/EWD10xx/EWD1036.html).
