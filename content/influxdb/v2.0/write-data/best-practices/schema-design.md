---
title: InfluxDB schema design
description: >
  Design your InfluxDB schema to reduce high cardinality and make your data more performant.
menu:
  influxdb_2_0:
    name: Schema design
    weight: 201
    parent: write-best-practices
---

Design your [schema](/influxdb/v2.0/reference/glossary/#schema) for simpler and more performant queries.
Follow design guidelines to make your schema easy to query.
Learn how these guidelines lead to more performant queries.

Store data in values
  Tag values
  Field values


Use simple measurements and keys
    Measurements
    Tag keys
    Field keys

Use tags to improve query performance

### Store data in values

Store data in [tag values](/influxdb/v2.0/reference/glossary/#tag-value) or [field values](/influxdb/v2.0/reference/glossary/#field-value), not in keys or [measurements](/influxdb/v2.0/reference/glossary/#measurement).

### Use one tag per attribute

Splitting a single tag with multiple pieces into separate tags simplifies your queries and improves performance by reducing the need for regular expressions.

#### Example line protocol schemas

Consider the following schema represented by line protocol.

```
Schema 1 - Multiple data encoded in a single tag
-------------
weather_sensor,crop=blueberries,location=plot-1.north temp=50.1 1472515200000000000
weather_sensor,crop=blueberries,location=plot-2.midwest temp=49.8 1472515200000000000
```

The Schema 1 data encodes multiple parameters, the `plot` and `region`, into a long tag value (`plot-1.north`).
Compare this to schema 2:

```
Schema 2 - Data encoded in multiple tags
-------------
weather_sensor,crop=blueberries,plot=1,region=north temp=50.1 1472515200000000000
weather_sensor,crop=blueberries,plot=2,region=midwest temp=49.8 1472515200000000000
```

Schema 2 is preferable because, with multiple tags, you don't need a regular expression.

#### Flux example to query schemas

The following Flux examples show how to calculate the average `temp` for blueberries in the `north` region; both for Schema 1 and Schema 2.

```js
// Schema 1 -  Query for multiple data encoded in a single tag
from(bucket:"example-bucket")
  |> range(start:2016-08-30T00:00:00Z)
  |> filter(fn: (r) =>  r._measurement == "weather_sensor" and r.location =~ /\.north$/ and r._field == "temp")
  |> mean()

// Schema 2 - Query for data encoded in multiple tags
from(bucket:"example-bucket")
  |> range(start:2016-08-30T00:00:00Z)
  |> filter(fn: (r) =>  r._measurement == "weather_sensor" and r.region == "north" and r._field == "temp")
  |> mean()
```
In Schema 1, we see that querying the `plot` and `region` in a single tag makes the data more difficult to query.




### Use fields for varying and numeric data
- Store numeric values in fields. ([Tags](/influxdb/v2.0/reference/glossary/#tag-value) only store strings).



### Use simple measurements and keys

Data stored in measurements and keys is more difficult to query and the queries are less performant.

  Measurements
  Tag keys
  Field keys

If you encode data in a [measurement](/influxdb/v2.0/reference/glossary/#measurement), [tag key](/influxdb/v2.0/reference/glossary/#tag-key), or [field key](/influxdb/v2.0/reference/glossary/#field-key), then you must use a regular expression when querying the data. Regular expressions make queries less efficient and more difficult to write.

{{% oss-only %}}

In addition, data encoded in measurements and keys increases their cardinality. This may further hurt performance because indexing assumes that measurements and field keys will have low cardinality.

{{% /oss-only %}}

#### Use simple measurements

A measurement is a simple name that describes your schema.
Use a different [measurement](/influxdb/v2.0/reference/glossary/#measurement) for each unique combination of [tag keys](/influxdb/v2.0/reference/glossary/#tag-key) and [field keys](/influxdb/v2.0/reference/glossary/#field-key).

Consider the following `air_sensor` and `water_quality_sensor` schemas:

| measurement          | tag key   | tag key  | field key | field key |
|----------------------|-----------|----------|-----------|-----------|
| air_sensor           | sensorId | location | pressure  | humidity  |
| water_quality_sensor | sensorId | location | pH        | salinity  |

The measurement describes the schema, the tags store common metadata, and the fields store variable numeric data.

#### Compare simple vs complex measurements

Consider the following schemas represented by line protocol.

**Recommended**: _Schema 1_ stores metadata in separate `crop`, `plot`, and `region` tags. The `temp` field contains variable numeric data.

```
Schema 1 - Data encoded in tags (recommended)
-------------
weather_sensor,crop=blueberries,plot=1,region=north temp=50.1 1472515200000000000
weather_sensor,crop=blueberries,plot=2,region=midwest temp=49.8 1472515200000000000
```

**Not recommended**: _Schema 2_ stores `plot` and `region` data (`blueberries.plot-1.north`) in the measurement, similar to Graphite metrics.

```
Schema 2 - Data encoded in the measurement (not recommended)
-------------
blueberries.plot-1.north temp=50.1 1472515200000000000
blueberries.plot-2.midwest temp=49.8 1472515200000000000
```

#### Compare measurement queries

For the same example schemas, use Flux to calculate the average `temp` for blueberries in the `north` region:

```js
// Schema 1 - Query data encoded in tags (recommended)
from(bucket:"example-bucket")
  |> range(start:2016-08-30T00:00:00Z)
  |> filter(fn: (r) =>  r._measurement == "weather_sensor" and r.region == "north" and r._field == "temp")
  |> mean()

// Schema 2 - Query data encoded in the measurement (not recommended)
from(bucket:"example-bucket")
  |> range(start:2016-08-30T00:00:00Z)
  |> filter(fn: (r) =>  r._measurement =~ /\.north$/ and r._field == "temp")
  |> mean()
```

To query Schema 2, you must use regular expressions to extract `plot` and `region` data from the measurement. The Schema 2 query is more difficult to write and less performant.

Complex measurements can make some queries impossible. For example, calculating the average temperature of both plots is not possible with _Schema 2_.




#### Use keys to label your columns
Follow these conventions when using tags and fields:

- [Avoid reserved keywords in tag and field keys](#avoid-reserved-keywords-in-tag-and-field-keys)
- [Avoid the same tag and field name](#avoid-the-same-name-for-a-tag-and-a-field)
- [Avoid encoding data in measurements and keys](#avoid-encoding-data-in-measurements-and-keys)

### Use tags to improve query performance

[Tag values](/influxdb/v2.0/reference/glossary/#tag-value) are indexed and [field values](/influxdb/v2.0/reference/glossary/#field-value) are not.
This means that querying by tags is more performant than querying by fields.

Your queries should guide what you store in tags and what you store in fields:
- Store data in tags if the values are used in [filter()]({{< latest "flux" >}}/universe/filter/) or [group()](/{{< latest "flux" >}}/universe/group/) functions.
- Store data in tags if the values are shared across multiple data points, i.e. it's metadata that describes a field value.
- Store unique values in fields.

However, as the number of indexes grows, both writes and reads may start to slow down.
To learn how series impact performance, see [avoid too many series](#avoid-too-many-series)

### Avoid keywords and special characters in keys

To simplify query writing, don't include reserved keywords or special characters in tag and field keys.
If you use [Flux keywords](/{{< latest "flux" >}}/spec/lexical-elements/#keywords) in keys,
then you'll have to wrap the keys in double quotes.
If you use non-alphanumeric characters in keys, then you'll have to use [bracket notation](/{{< latest "flux" >}}/data-types/composite/record/#bracket-notation) in Flux.

### Avoid the same name for a tag and a field

Avoid using the same name for a [tag key](/influxdb/v2.0/reference/glossary/#tag-key) and a [field key](/influxdb/v2.0/reference/glossary/#field-key), which may produce unexpected results when querying data.

{{% cloud-only %}}

{{% note %}}
Use [explicit bucket schemas]() to enforce unique tags and field names within a schema.
{{% /note %}}

{{% /cloud-only %}}



### Avoid putting more than one piece of information in one tag


## Avoid too many series

{{% oss-only %}}

  IndexDB indexes the following data elements to speed up reads:
  - [measurement](/influxdb/v2.0/reference/glossary/#measurement)
  - [tags](/influxdb/v2.0/reference/glossary/#tag)

{{% /oss-only %}}
{{% cloud-only %}}

  IndexDB indexes the following data elements to speed up reads:
  - [measurement](/influxdb/v2.0/reference/glossary/#measurement)
  - [tags](/influxdb/v2.0/reference/glossary/#tag)
  - [field keys](/influxdb/cloud/reference/glossary/#field-key)

{{% /cloud-only %}}

{{% oss-only %}}

  [`SHOW SERIES CARDINALITY`](/influxdb/v2.0/query_language/spec/#show-series-cardinality) measures the number of unique [series](/influxdb/v2.0/reference/glossary/#series) in your data.

{{% /oss-only %}}

{{% cloud-only %}}

  [`influxdb.cardinality()`](/{{< latest "flux" >}}/stdlib/influxdata/influxdb/cardinality) measures the number of series keys in your data.

{{% /cloud-only %}}


Each unique set of indexed data elements forms a [series key](/influxdb/v2.0/reference/glossary/#series-key).
[Tags](/influxdb/v2.0/reference/glossary/#tag) containing highly variable information like unique IDs, hashes, and random strings lead to a large number of [series](/influxdb/v2.0/reference/glossary/#series), also known as high [series cardinality](/influxdb/v2.0/reference/glossary/#series-cardinality).
High series cardinality is a primary driver of high memory usage for many database workloads.
Therefore, to reduce memory overhead, consider storing high-cardinality values in fields rather than in tags.

{{% note %}}

If reads and writes to InfluxDB start to slow down, you may have high series cardinality (too many series).
See how to [resolve high cardinality](/influxdb/v2.0/write-data/best-practices/resolve-high-cardinality/).

{{% /note %}}

<!--
## Shard group duration management

InfluxDB stores data in shard groups.
Shard groups are organized by [buckets](/influxdb/v2.0/reference/glossary/#bucket) and store data with timestamps that fall within a specific time interval called the [shard duration](/influxdb/v1.8/concepts/glossary/#shard-duration).

If no shard group duration is provided, the shard group duration is determined by the RP [duration](/influxdb/v1.8/concepts/glossary/#duration) at the time the RP is created. The default values are:

| RP Duration  | Shard Group Duration  |
|---|---|
| < 2 days  | 1 hour  |
| >= 2 days and <= 6 months  | 1 day  |
| > 6 months  | 7 days  |

The shard group duration is also configurable per RP.
To configure the shard group duration, see [Retention Policy Management](/influxdb/v1.8/query_language/manage-database/#retention-policy-management).

### Shard group duration tradeoffs

Determining the optimal shard group duration requires finding the balance between:

- Better overall performance with longer shards
- Flexibility provided by shorter shards

#### Long shard group duration

Longer shard group durations let InfluxDB store more data in the same logical location.
This reduces data duplication, improves compression efficiency, and improves query speed in some cases.

#### Short shard group duration

Shorter shard group durations allow the system to more efficiently drop data and record incremental backups.
When InfluxDB enforces an RP it drops entire shard groups, not individual data points, even if the points are older than the RP duration.
A shard group will only be removed once a shard group's duration *end time* is older than the RP duration.

For example, if your RP has a duration of one day, InfluxDB will drop an hour's worth of data every hour and will always have 25 shard groups. One for each hour in the day and an extra shard group that is partially expiring, but isn't removed until the whole shard group is older than 24 hours.

>**Note:** A special use case to consider: filtering queries on schema data (such as tags, series, measurements) by time. For example, if you want to filter schema data within a one hour interval, you must set the shard group duration to 1h. For more information, see [filter schema data by time](/influxdb/v1.8/query_language/explore-schema/#filter-meta-queries-by-time).

### Shard group duration recommendations

The default shard group durations work well for most cases. However, high-throughput or long-running instances will benefit from using longer shard group durations.
Here are some recommendations for longer shard group durations:

| RP Duration  | Shard Group Duration  |
|---|---|
| <= 1 day  | 6 hours  |
| > 1 day and <= 7 days  | 1 day  |
| > 7 days and <= 3 months  | 7 days  |
| > 3 months  | 30 days  |
| infinite  | 52 weeks or longer  |

> **Note:** Note that `INF` (infinite) is not a [valid shard group duration](/influxdb/v1.8/query_language/manage-database/#retention-policy-management).
In extreme cases where data covers decades and will never be deleted, a long shard group duration like `1040w` (20 years) is perfectly valid.

Other factors to consider before setting shard group duration:

* Shard groups should be twice as long as the longest time range of the most frequent queries
* Shard groups should each contain more than 100,000 [points](/influxdb/v1.8/concepts/glossary/#point) per shard group
* Shard groups should each contain more than 1,000 points per [series](/influxdb/v1.8/concepts/glossary/#series)

#### Shard group duration for backfilling

Bulk insertion of historical data covering a large time range in the past creates a large number of shards at once.
The concurrent access and overhead of writing to hundreds or thousands of shards can quickly lead to slow performance and memory exhaustion.

When writing historical data, consider your ingest rate limits, volume, and existing data schema affects performance and memory.

-->
