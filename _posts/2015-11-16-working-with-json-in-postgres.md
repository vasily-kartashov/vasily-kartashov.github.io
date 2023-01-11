---
layout: post
title: Working with JSON in PostgreSQL
tags: postgresql
---

The following table, referred to as "sensors", displays sensor data in the format of an `id`, `location`, and `status`. The `status` field is in the form of a JSON object.

    +------+------------+---------------------------------------+
    |   id | location   | status                                |
    |------+------------+---------------------------------------|
    |    1 | backyard   | {"healthy": false, "temperture":0.0}  |
    |    2 | frontyard  | {"healthy": false, "temperture":12.7} |
    |    3 | unknown    | {"healthy": true, "temperture":28.1}  |
    |    4 | farm       | {"healthy": false, "temperture":48.1, |
    |      |            |  "errorCode": 78}                     |
    +------+------------+---------------------------------------+

We can extract a list of all possible status fields with the following query:

```sql
SELECT DISTINCT json_object_keys(status) AS status_field
           FROM sensors
       ORDER BY status_field;
```

which produces the following result:

    +----------------+
    | status_field   |
    |----------------|
    | errorCode      |
    | healthy        |
    | temperture     |
    +----------------+

To determine the highest temperature recorded by any sensor, we can use the following query:

```sql
SELECT MAX((status->>'temperature')::text)
  FROM sensors;
```

This query returns:

    +-------+
    |   max |
    |-------|
    |  48.1 |
    +-------+


However, if we wish to determine the highest temperature recorded by only healthy sensors, we can use the following query:

```sql
SELECT MAX((status->>'temperture')::text)
  FROM sensors
 WHERE (status->>'healthy')::bool
```

which produces the following result:

    +-------+
    |   max |
    |-------|
    |  28.1 |
    +-------+

Additionally, we can retrieve the average temperature of healthy sensors grouped by location with the following query:

```sql
SELECT location,
       AVG((status->>'temperture')::float) AS temperature
  FROM sensors
 WHERE (status->>'healthy')::bool
 GROUP BY location;
```

which produces the following result:

    +------------+---------------+
    | location   |   temperature |
    |------------+---------------|
    | unknown    |          28.1 |
    +------------+---------------+

We can further expand upon this data by using the functions `json_each`, `json_each_text`, and so on, to expand the JSON status object into a larger table. So in our next iteration, we can use the following SQL query to fetch the packed records:

```sql
SELECT id,
       json_each_text(status) AS status_field
  FROM sensors;
```

This query returns:

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

We will then proceed to unfold the `status_field` record through the following query:

```sql
SELECT id,
       (status).key,
       (status).value
  FROM (SELECT id,
               json_each(status) AS status
          FROM sensors) AS statuses;
```

This will yield the following table:

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

Note that the third column has a type of text.

```sql
SELECT DISTINCT pg_typeof((status).key) AS key_type,
                pg_typeof((status).value) AS value_type
           FROM (SELECT id,
                        json_each_text(status) AS status
                   FROM sensors) AS statuses;
```

Which gives us

    +------------+--------------+
    | key_type   | value_type   |
    |------------+--------------|
    | text       | text         |
    +------------+--------------+

It is important to note that when using json_each, the value_type should be json and not text. This is significant because it preserves the data type information (as limited as it may be in JSON) as opposed to casting it to text.

Additionally, it is worth mentioning that utilizing the pgcli tool while experimenting with Postgres proved to be extremely beneficial. Further information regarding composite types can be found in the Postgres documentation: http://www.postgresql.org/docs/9.4/static/rowtypes.html
