---
layout: post
title: Writing a simple query language with ANTLR
---

Let's say we have a complex database of IoT devices and we want to enable a simple search interface that would allow us to send queries like following


    location within 10 km from (-37.814, 144.963) and status.stateOfCharge < 10%


Although it might sound intimidatingly complex task, the fact is it's rather simple to achive with ANTLR. Let's begin with the query language.