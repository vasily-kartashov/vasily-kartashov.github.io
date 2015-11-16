---
layout: post
title: Working with JSON in PostgreSQL
---

The original table sensors looks like following

    +------+------------+---------------------------------------+
    |   id | location   | status                                |
    |------+------------+---------------------------------------|
    |    1 | backyard   | {"healthy": false, "temperture":0.0}  |
    |    2 | frontyard  | {"healthy": false, "temperture":12.7} |
    |    3 | unknown    | {"healthy": true, "temperture":28.1}  |
    |    4 | farm       | {"healthy": false, "temperture":48.1, |
    |      |            |  "errorCode": 78}                     |
    +------+------------+---------------------------------------+

Let's find all possible status fields

{% highlight sql %}
SELECT DISTINCT json_object_keys(status) AS status_field
           FROM sensors
       ORDER BY status_field;
{% endhighlight %}

gives us the following list

    +----------------+
    | status_field   |
    |----------------|
    | errorCode      |
    | healthy        |
    | temperture     |
    +----------------+

Now let's find the highest temperature read by a unit

{% highlight sql %}
SELECT MAX((status->>'temperture')::text)
  FROM sensors;
{% endhighlight %}

give us

    +-------+
    |   max |
    |-------|
    |  48.1 |
    +-------+


What if we want to know the highest temperature measured by a healthy unit?

{% highlight sql %}
SELECT MAX((status->>'temperture')::text)
  FROM sensors
 WHERE (status->>'healthy')::bool
{% endhighlight %}

produces a different picture

    +-------+
    |   max |
    |-------|
    |  28.1 |
    +-------+

Get the average temperature from the healthy sensors grouped by location

{% highlight sql %}
SELECT location,
       AVG((status->>'temperture')::float) AS temperature
  FROM sensors
 WHERE (status->>'healthy')::bool
 GROUP BY location;
{% endhighlight %}

Gives you this very short list

    +------------+---------------+
    | location   |   temperature |
    |------------+---------------|
    | unknown    |          28.1 |
    +------------+---------------+

Let's unwrap the key value store from status into a bigger table. The unfolding operations such as `json_each`, `json_each_text` and so on produce sets of records. In our first iteration let's fetch the packed records.

{% highlight sql %}
SELECT id,
       json_each_text(status) AS status_field
  FROM sensors;
{% endhighlight %}

which gives us the following table

    +------+-------------------+
    |   id | json_each_text    |
    |------+-------------------|
    |    1 | (healthy,false)   |
    |    1 | (temperture,0.0)  |
    |    2 | (healthy,false)   |
    |    2 | (temperture,12.7) |
    |    3 | (healthy,true)    |
    |    3 | (temperture,28.1) |
    |    4 | (healthy,false)   |
    |    4 | (temperture,48.1) |
    |    4 | (errorCode,78)    |
    +------+-------------------+

And now let's unfold the `status_field` record

{% highlight sql %}
SELECT id,
       (status).key,
       (status).value
  FROM (SELECT id,
               json_each(status) AS status
          FROM sensors) AS statuses;
{% endhighlight %}

Which gives us the following table

    +------+------------+---------+
    |   id | key        | value   |
    |------+------------+---------|
    |    1 | healthy    | false   |
    |    1 | temperture | 0.0     |
    |    2 | healthy    | false   |
    |    2 | temperture | 12.7    |
    |    3 | healthy    | true    |
    |    3 | temperture | 28.1    |
    |    4 | healthy    | false   |
    |    4 | temperture | 48.1    |
    |    4 | errorCode  | 78      |
    +------+------------+---------+

Here's the relevant documentation from Postgres website regarding the composite types: http://www.postgresql.org/docs/9.4/static/rowtypes.html.
Note that the third column has a type of text.

{% highlight sql %}
SELECT DISTINCT pg_typeof((status).key) AS key_type,
                pg_typeof((status).value) AS value_type
           FROM (SELECT id,
                        json_each_text(status) AS status
                   FROM sensors) AS statuses;
{% endhighlight %}

Which gives us

    +------------+--------------+
    | key_type   | value_type   |
    |------------+--------------|
    | text       | text         |
    +------------+--------------+

When using `json_each` the `value_type` whould be `json` and not `text` which is important as this means data type information (as scarse it is in JSON) is not completely lost by casting it to text.

Last but not least: it was a hugely beneficial to use `pgcli` while experimenting with Postgres.