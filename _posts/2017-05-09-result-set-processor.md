---
layout: post
title: Result Set Processor
---

Join tables and get data in one go, but split and group and convert manually, or fetch only the data that you need at the moment and face N+1 kind of problems? Lets assume we have a list of `controllers` were each has a set of `sensors` attached. The result should look something like

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

And here are two issues, do you go with

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

or

```
foreach (SELECT controllers.id FROM controllers => $id) {
    SELECT * FROM sensors WHERE sensors.controller_id = $id;
}

```

First solution although idiomatic is also tedious and can grow unhealthy for more complex use cases. The second one has body of literature dedicated to it, which is never a good sign.

The rest of the exposition is in PHP. Bear with me.

The job of processing result sets is actually an interesting domain. The underlying data structure is a set of records.

```
abstract class ProcessingContext
{
    private $records;
}
```

If there are no informaition about the structure of these records, then the only two meaningful operations for this kind of context are `collectAll` and `collectHead`. I'll call this context `Collector`

```
class Collector extends ProcessingContext
{
    public function collectAll(): array ...

    public function collectHead() ...

}
```

If we know that the `$records` are associative arrays, then we have more interesting things to do. But whatever we have in mind with this records it should always start with picking fields from each record for further processing. So the next context is going to be a Selector. Clearly whilst knowing that `records` are associative arrays helps us with adding more operations accessible from this context, we still can `collectHead` and `collectAll`, so we a derived class:

```
class Selector extends Collector
{
    public function selectValue(string $field): Converter ...

    public function selectFields(strings... $fields): Converter ...

    public function map(string $keyField, string $valueField): MapConverter ...
}
```

What is a `Converter` or `MapConverter`? A selector lets you pick fields from each record and place them in some sort of a structure. For example `selectValue` let's you pick a value of a field and store it as a scalar, `selectFields` let's you fetch an embedded associative array (subrecord if you wish) from each record, and `map` lets you create a new associative array from values of two fields. The `Converter` is the context in which the API user needs to decide what to do with the selected subrecord.

```
class Converter
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

So method `name` returns the subrecord back into the record it was extracted from under new name. Method `group` groups subrecords by using the remainder of each record as group name. It doesn't return the group back into the record, so the result is actually a collector, i.e. the records of the resulting collector are the groups. The `groupInto` on the other side not only groups subrecords, but also returns the groups back into the record.

At this point I expect the reader to lose all interest. Here is how I would split the example join above

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

That is literally all. You can take a look at the [example imeplementation](https://github.com/vasily-kartashov/hamlet-core/tree/version-2.1/src/Hamlet/Database/Processing) and some [unit tests](https://github.com/vasily-kartashov/hamlet-core/blob/version-2.1/tests/Hamlet/Database/ProcessorTest.php).
