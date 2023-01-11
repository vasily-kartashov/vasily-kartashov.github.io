---
layout: post
title: Working with JSON in Postgres 9.5
tags: postgresql
---

Docker is an incredibly useful tool that allows for easy experimentation with new software. It enables the simultaneous installation, suspension, and deletion of multiple versions of programs, such as Postgres, with minimal effort. The installation process is as simple as executing the following command:

```bash
sudo apt-get install docker.io
sudo docker run --name postgres -e POSTGRES_PASSWORD=postgres -d postgres:9.5
```

Once Docker is installed, you can access an interactive Postgres shell by executing the following command:

```bash
sudo docker exec -it postgres psql -U postgres
```

With this tool, we can now delve into the new JSON functionality available in Postgres 9.5. We will begin by creating two tables, devices and observations, and adding some data to them:

```sql
CREATE TABLE sensors (
    id     integer PRIMARY KEY,
    type   character varying(64) NOT NULL,
    config jsonb NOT NULL
);

CREATE INDEX config_gin ON sensor USING gin(config);

INSERT INTO sensors (id, type, config)
     VALUES (1, 'soil-moisture',    '{ "alpha": 0.543, "beta": -2.312, "enabled": true }'),
            (2, 'soil-temperature', '{ "enabled": true, "depth": 0.24 }'),
            (3, 'humidity', '{ "enabled": false, "height": 1.34, "device": { "version": "3.4", "supported": true } }');
```

We can then enable the humidity sensor:

```sql
UPDATE sensors
   SET config = config || '{ "enabled": true }'::jsonb
 WHERE id = 3;
```

Remove the `alpha` and the `beta` parameters used for soil moisture calibration:

```sql
UPDATE sensors
   SET config = config - 'alpha' - 'beta'
 WHERE type = 'soil-moisture';
```

And now let's remove the supported tag from all device sections:

```sql
UPDATE sensors
   SET config = config #- '{device,supported}';
```

Fetch the device version information wherever it's available:

```sql
WITH versioning AS (
    SELECT id, type, config #>> '{device,version}' AS version
      FROM sensors
)
SELECT *
  FROM versioning
 WHERE version IS NOT NULL;
```

Find all the sensors where the depth is specified:

```sql
SELECT *
  FROM sensors
 WHERE config ? 'depth';
```

Let's find all properly placed sensors, where either the `depth` or the `height` are specified, but not both:

```sql
SELECT *
  FROM sensors
 WHERE config ?| array['depth', 'height']
   AND NOT config ?& array['depth', 'height'];
```

For further reading on the JSON functionality available in Postgres 9.5, I recommend consulting the [official documentation](http://www.postgresql.org/docs/9.5/static/functions-json.html).

