# Learn About Redpanda and Materialize integration

While the goal of this project is to drive Materialize through dbt models, it's useful to
have a basic understanding of how Redpanda and Materialize are integrated. Redpanda is fully compatible
with the Kafka protocol, and can hook into Materialize's Kafka integraiton. In this section,
you will learn how to:
- Create a Redpanda topic
- Produce data to a Redpanda topic
- Create a SOURCE for Materialize
- Create VIEWs and query the topic data

First, we will create a Redpanda topic and send data from the file `data/icao24_mapping_airbus.json.gz`
into that topic. From the base directory `sample_project`, run the following command to create a Redpanda topic.

```bash
docker compose exec redpanda rpk topic create icao24_mapping
```

Send the decompressed contents of the data file to stdout, and pipe that to `rpk topic produce` which
sends each line as a message to the `icao24_mapping` topic.

```bash
gunzip -c data/icao24_mapping_airbus.json.gz | docker compose exec redpanda rpk topic produce icao24_mapping
```

You should a bunch of messages like:
```
...
Produced to partition 0 at offset 10133 with timestamp 1644078989367.
Produced to partition 0 at offset 10134 with timestamp 1644078989367.
Produced to partition 0 at offset 10135 with timestamp 1644078989367.
Produced to partition 0 at offset 10136 with timestamp 1644078989367.
Produced to partition 0 at offset 10137 with timestamp 1644078989367.
```

Check the messages in the topic by consuming them.
This will continuously poll the topic, until you press `CTRL-C` to break out.

```
docker compose exec redpanda rpk topic consume icao24_mapping
```

You should see something like the following.
```json
{
  "topic": "icao24_mapping",
  "value": "{\"icao24\":\"4ba990\",\"manufacturericao\":\"AIRBUS\",\"manufacturername\":\"Airbus\",\"model\":\"Airbus A319 132\",\"typecode\":\"A319\",\"icaoaircrafttype\":\"L2J\",\"operator\":\"\",\"operatorcallsign\":\"TURKAIR\",\"operatoricao\":\"THY\",\"built\":\"\",\"categorydescription\":\"No ADS-B Emitter Category Information\"}",
  "timestamp": 1644078989367,
  "partition": 0,
  "offset": 10130
}
{
  "topic": "icao24_mapping",
  "value": "{\"icao24\":\"a7e62d\",\"manufacturericao\":\"AIRBUS\",\"manufacturername\":\"Airbus\",\"model\":\"A320-232\",\"typecode\":\"A320\",\"icaoaircrafttype\":\"L2J\",\"operator\":\"Spirit Airlines\",\"operatorcallsign\":\"SPIRIT WINGS\",\"operatoricao\":\"NKS\",\"built\":\"2011-01-01\",\"categorydescription\":\"No ADS-B Emitter Category Information\"}",
  "timestamp": 1644078989367,
  "partition": 0,
  "offset": 10131
}
```

At this point, you know how to create a topic, and how to send and receive data.
You can proceed to creating SOURCEs and VIEWs in dbt models, or you can check that
your Redpanda integration works via the Materialize CLI.

Fire up the Materialize CLI.

```bash
docker-compose run mzcli
```

Create a MATERIALIZED SOURCE for the topic.

```sql
CREATE MATERIALIZED SOURCE tmp_icao24_topic
  FROM KAFKA BROKER 'redpanda:9092' TOPIC 'icao24_mapping'
    WITH (timeline = 'user')
  FORMAT TEXT ;
```

Create a VIEW for that topic that converts JSON to flat tabular format.

```sql
CREATE VIEW tmp_icao24_flat AS (
  SELECT
    (data->>'icao24')::string               AS icao24,
    (data->>'manufacturericao')::string     AS manufacturericao,
    (data->>'manufacturername')::string     AS manufacturername,
    (data->>'model')::string                AS model,
    (data->>'typecode')::string             AS typecode,
    (data->>'icaoaircrafttype')::string     AS icaoaircrafttype,
    (data->>'operator')::string             AS operator,
    (data->>'operatorcallsign')::string     AS operatorcallsign,
    (data->>'operatoricao')::string         AS operatoricao,
    (data->>'built')::string                AS built,
    (data->>'categorydescription')::string  AS categorydescription
  FROM (
    SELECT CAST (text AS JSONB) AS data
    FROM tmp_icao24_topic
  )
) ;
```

Run a query.

```sql
select icao24, typecode, operator from tmp_icao24_flat limit 10 ;
```

You should see something like the following.

```sql
 icao24 | typecode | operator
--------+----------+-----------
 38405a | A388     |
 3840ba | A388     |
 3c0d50 | A320     |
 3f6219 | A400     |
 407b55 | A321     |
 4cc528 | A21N     |
 e80326 | A20N     | Sku
 e80328 | A20N     | Sku
 01022c | A20N     | Air Cairo
 01022d | A20N     | Air Cairo
(10 rows)
```

It works! Optionally, you can clean up after yourself:

```bash
DROP SOURCE IF EXISTS tmp_icao24_topic ;
DROP VIEW IF EXISTS tmp_icao24_flat ;
```
