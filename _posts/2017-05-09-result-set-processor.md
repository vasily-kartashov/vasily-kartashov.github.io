---
layout: post
title: Result Set Processor
tags: php
---

When it comes to fetching data from a database, there's often a dilemma of whether to join tables and retrieve all the data in one go, or fetch only the necessary data for the moment and risk N+1 query problems. Let's assume we have a list of controllers, each with a set of sensors attached. The result should look something like:

```
[
    {
        "controllerId": "c001",
        "sensors": []
    },
    {
        "controllerId": "c002",
        "sensors": [
            {
                "sensorId": "s001",
                "active": false
            }
        ]
    }
]
```

One solution is to use the following SQL query:

```
SELECT controllers.id AS controller_id,
       sensors.id AS sensor_id
       sensors.active
  FROM controllers
       JOIN sensors
         ON contoller.id = sensors.controller_id;

$controller = null;
foreach ($tupes as $tuple) {
    if ($tuple.controller != $controller) {
        $controller = $tuple.controller;
    }
    $controller.append($tuple.sensor);
}
```

However, this can become tedious and complex for more advanced use cases. Another solution is to use a foreach loop and fetch data for each controller individually:

```
foreach (SELECT controllers.id FROM controllers => $id) {
    SELECT * FROM sensors WHERE sensors.controller_id = $id;
}

```

This second solution also has its downsides and can lead to N+1 query problems.

In this article, the author presents a way to address this dilemma in PHP by using the concept of processing contexts. The basic idea is to define a `ProcessingContext` class that keeps track of a set of records and provides methods for manipulating and extracting those records.

The job of processing result sets is a fascinating area to explore. The core data structure is a set of records:

```
abstract class ProcessingContext
{
    private $records;
}
```

When there is no information about the structure of these records, the only meaningful operations for this context are `collectAll` and `collectHead`. This context can be called a `Collector`:

```
class Collector extends ProcessingContext
{
    public function collectAll(): array ...

    public function collectHead() ...

}
```

Once we know that the $records are associative arrays, we can do more interesting things with them. However, whatever we want to do with these records, it should always start with picking a subset of each record for further processing. The next context is the `Selector`. Even though knowing that the records are associative arrays allows us to add more operations to the context, we can still do what the `Collector` does i.e. `collectHead` and `collectAll`:

```
class Selector extends Collector
{
    public function selectValue(string $field): Converter ...

    public function selectFields(strings... $fields): Converter ...

    public function map(string $keyField, string $valueField): MapConverter ...
}
```

What is a `Converter` or a `MapConverter`? A selector allows you to pick fields from each record and place them in some sort of structure. For example, `selectValue` lets you pick a value of a field and store it as a scalar, `selectFields` lets you fetch an embedded associative array from each record, and map lets you create a new key/value pair from the values of two fields. The `Converter` is the context in which the API user must decide what to do with the selected subrecord.

```
class Converter extends ProcessingContext
{
    public function name(string $name): Selector ...

    public function group(): Collector ...

    public function groupInto(string $name): Selector ...

    ...
}

class MapConverter extends Converter
{
    public function flatten(): Collector ...

    public function flattenInto(string $name): Selector ...

    ...
}
```

So the `name` method returns the subrecord back into the record it was extracted from under a new name. The `group` method groups subrecords by using the remainder of each record as a group key. It does not return the group back into the record, so the result of `group` is actually a collector, i.e. the records are the groups extracted by the selector. The `groupInto` method not only groups subrecords but also pushes the groups back into the record.

I understand that the example provided may be complex and difficult to follow. Here is how I would simplify the example join above:

```
$query = "
    SELECT controllers.id,
           sensors.id AS sensor_id
           sensors.active AS sensor_active
      FROM controllers
           JOIN sensors
             ON contoller.id = sensors.controller_id;
";
$procedure = $database->prepare($query);
$procedure->processAll()
    ->selectByPrefix('sensor_')->group('sensors')
    ->collectAll();
```

The records would look like this:

```
| id   | sensor_id | sensor_active |
+------+-----------+---------------+
| c001 | NULL      | NULL          |
| c002 | s001      | false         |
```

Then, we select by prefix and group them into a record called sensors:

```
| id   | sensors                       |
+------+-------------------------------+
| c001 | []                            |
| c002 | [{ id: s001, active: false }] |
```

And that's it! If you'd like oto see a working example, you can check out the [example imeplementation](https://github.com/vasily-kartashov/hamlet-core/tree/version-2.1/src/Hamlet/Database/Processing) and some [unit tests](https://github.com/vasily-kartashov/hamlet-core/blob/version-2.1/tests/Hamlet/Database/ProcessorTest.php) for further reference.
