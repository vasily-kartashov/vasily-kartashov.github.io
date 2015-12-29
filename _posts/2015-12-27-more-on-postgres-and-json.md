---
layout: post
title: Working with JSON in Postgres 9.5
---

Let's start with installing docker, which is a very convenient tool for the cases where you just want to try out new software. For example,
you can have multiple versions of Postgres running on your computer, which you can install, suspend or delete completely without any trace with just a few commands.
The installation command looks like following:

{% highlight bash %}
sudo apt-get install docker.io
sudo docker run --name postgres -e POSTGRES_PASSWORD=postgres -d postgres:9.5
{% endhighlight %}

Now open an Postgres' interactive shell by executing

{% highlight bash %}
sudo docker exec -it postgres psql -U postgres
{% endhighlight %}

You are in, so we can start with exploring new JSON functionality available in Postgres 9.5. Let's create two tables: `devices` and `observations`.

{% highlight sql %}
CREATE TABLE sensors (
    id     integer PRIMARY KEY,
    type   character varying(64) NOT NULL,
    config jsonb NOT NULL
);

INSERT INTO sensors (id, type, config)
     VALUES (1, 'soil-moisture',    '{ "alpha": 0.543, "beta": -2.312, "enabled": true }'),
            (2, 'soil-temperature', '{ "enabled": true, "depth": 0.24 }'),
            (3, 'humidity', '{ "enabled": false, "height": 1.34, "device": { "version": "3.4", "supported": true } }');
{% endhighlight %}

Let's enable the humidity sensor

{% highlight sql %}
UPDATE sensors
   SET config = config || '{ "enabled": true }'::jsonb
 WHERE id = 3;
{% endhighlight %}

And let's remove the `alpha` and the `beta` parameters used for soid moisture calibration

{% highlight sql %}
UPDATE sensors
   SET config = config - 'alpha' - 'beta'
 WHERE type = 'soil-moisture';
{% endhighlight %}

And now let's remove the supported tag from all `device` sections
{% highlight sql %}
UPDATE sensors
   SET config = config #- '{device,supported}';
{% endhighlight %}

Let's fetch the device version information wherever it's available

{% highlight sql %}
WITH versioning AS (
    SELECT id, type, config #>> '{device,version}' AS version
      FROM sensors
)
SELECT *
  FROM versioning
 WHERE version IS NOT NULL;
{% endhighlight %}

Find all the sensors where the `depth` is specified
{% highlight sql %}
SELECT *
  FROM sensors
 WHERE config ? 'depth';
{% endhighlight %}

Let's find all properly placed sensors, where either the `depth` or the `height` are specified, but not both
{% highlight sql %}
SELECT *
  FROM sensors
 WHERE config ?| array['depth', 'height']
   AND NOT config ?& array['depth', 'height'];
{% endhighlight %}

You can read more on JSON functionality in Postgres 9.5 in the [official documentation](http://www.postgresql.org/docs/9.5/static/functions-json.html)