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
  ### Use tags to improve query performance
  ### Design your tags
  ### Use fields to store fluctuating and numeric data

Use simple measurements and keys
    Design your measurements
    Design your keys

Good schema design results in better performing queries by avoiding high series cardinality. If you notice data reads and writes slowing down or want to learn how cardinality affects performance, see how to [resolve high cardinality](/influxdb/v2.0/write-data/best-practices/resolve-high-cardinality/).

### Store data in values

Store data in [tag values](/influxdb/v2.0/reference/glossary/#tag-value) or [field values](/influxdb/v2.0/reference/glossary/#field-value), not in keys or [measurements](/influxdb/v2.0/reference/glossary/#measurement).

Data stored in values is easier and more performant to query. Data encoded in names (measurements, tag keys, and field keys) is harder and less efficient to query.

### Use tags to improve query performance

[Tag values](/influxdb/v2.0/reference/glossary/#tag-value) are indexed and [field values](/influxdb/v2.0/reference/glossary/#field-value) are not.
This means that querying tags is more performant than querying fields.

Your queries should guide what you store in tags and what you store in fields:
- Store values as [tag values](/influxdb/v2.0/reference/glossary/#tag-value) if the values are used in [filter()]({{< latest "flux" >}}/universe/filter/) or [group()](/{{< latest "flux" >}}/universe/group/) functions.
- Store values as tag values if the values are shared across multiple data points, i.e. metadata about the field value.

See how to [design your tags](#design-your-tags).

### Design your tags

#### Use one tag per attribute

If your source data contains multiple data attributes in a single parameter,
split each attribute into its own tag or field.
When each tag has only one concern, not multiple concatenated attributes,
you can write simpler queries and improve performance by reducing the need for regular expressions.

#### Compare schemas

Consider the following valid schemas represented by line protocol.

**Not recommended**: data in [_Bad Tags_](#bad-tags-schema) schema encodes multiple parameters (`plot` and `region`) in a single `location` tag value (`plot-1.north`).

##### {id="bad-tags-schema"}
```
Bad Tags schema - Multiple data encoded in a single tag
-------------
weather_sensor,crop=blueberries,location=plot-1.north temp=50.1 1472515200000000000
weather_sensor,crop=blueberries,location=plot-2.midwest temp=49.8 1472515200000000000
```

**Recommended**: [_Good Tags_](#good-tags-schema) schema splits the location data into separate `plot` and `region` tags.

#####  {id="good-tags-schema"}
```
Good Tags schema - Data encoded in multiple tags
-------------
weather_sensor,crop=blueberries,plot=1,region=north temp=50.1 1472515200000000000
weather_sensor,crop=blueberries,plot=2,region=midwest temp=49.8 1472515200000000000
```

#### Compare queries

Compare queries of the same [_Good Tags_](#good-tags-schema) and [_Bad Tags_](#bad-tags-schema) schemas.
The following Flux queries calculate the average `temp` for blueberries in the `north` region.

**Hard to query**: requires regular expressions to parse the complex `location` values of [_Bad Tags_](#bad-tags-schema).

```js
// Query *Bad Tags* schema for multiple data encoded in a single tag
from(bucket:"example-bucket")
  |> range(start:2016-08-30T00:00:00Z)
  |> filter(fn: (r) =>  r._measurement == "weather_sensor" and r.location =~ /\.north$/ and r._field == "temp")
  |> mean()
```

**Easy to query**: doesn't require regular expressions to query the `region` in [_Good Tags_](#good-tags-schema).

```js
// Query *Good Tags* schema for data encoded in multiple tags
from(bucket:"example-bucket")
  |> range(start:2016-08-30T00:00:00Z)
  |> filter(fn: (r) =>  r._measurement == "weather_sensor" and r.region == "north" and r._field == "temp")
  |> mean()
```

### Use fields to store fluctuating and numeric data

- Store unique or frequently changing values as field values.
- Store numeric values as field values. ([Tags](/influxdb/v2.0/reference/glossary/#tag-value) only store strings).

### Use simple measurements and keys

- [Use simple measurements](#use-simple-measurements)
- [Use simple keys](#use-simple-keys)

Consider the following `air_sensor` and `water_quality_sensor` schemas:

| measurement          | tag key   | tag key  | field key | field key |
|----------------------|-----------|----------|-----------|-----------|
| air_sensor           | sensorId  | location | pressure  | humidity  |
| water_quality_sensor | sensorId  | location | pH        | salinity  |

- Each measurement describes a unique schema.
- Tag keys and field keys are simple column names.
- Tags (`sensorId` and `location`) store metadata common across many data points.
- Fields (`humidity`, `pressure`, `pH`, and `salinity`) store numeric data.
- Fields store data that is unique or frequently changing.

In the example schemas above, the names (measurements and keys) don't contain data.
If you encode data in a [measurement](/influxdb/v2.0/reference/glossary/#measurement), [tag key](/influxdb/v2.0/reference/glossary/#tag-key), or [field key](/influxdb/v2.0/reference/glossary/#field-key), then you must use a regular expression when querying the data. Regular expressions make queries less efficient and more difficult to read and write.

{{% oss-only %}}

In addition, data encoded in measurements and keys can lead to decreased performance due to [higher series cardinality](/influxdb/v2.0/write-data/best-practices/resolve-high-cardinality/).

{{% /oss-only %}}

#### Design your measurements

#### Compare schemas

Consider the following valid schemas represented by line protocol.

**Recommended**: _Schema 1_ stores metadata in separate `crop`, `plot`, and `region` tags. The `temp` field contains variable numeric data.

```
Good schema - Data encoded in tags (recommended)
-------------
weather_sensor,crop=blueberries,plot=1,region=north temp=50.1 1472515200000000000
weather_sensor,crop=blueberries,plot=2,region=midwest temp=49.8 1472515200000000000
```

**Not recommended**: _Schema 2_ stores `plot` and `region` data (`blueberries.plot-1.north`) in the measurement, similar to Graphite metrics.

```
Bad schema - Data encoded in the measurement (not recommended)
-------------
blueberries.plot-1.north temp=50.1 1472515200000000000
blueberries.plot-2.midwest temp=49.8 1472515200000000000
```

#### Compare queries

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


#### Design your keys

Follow these guidelines when naming tags and fields:

- [Avoid reserved keywords in tag and field keys](#avoid-reserved-keywords-in-tag-and-field-keys)
- [Avoid the same tag and field name](#avoid-the-same-name-for-a-tag-and-a-field)

### Avoid keywords and special characters in keys

To simplify query writing, don't include reserved keywords or special characters in tag and field keys.
If you use [Flux keywords](/{{< latest "flux" >}}/spec/lexical-elements/#keywords) in keys,
then you'll have to wrap the keys in double quotes.
If you use non-alphanumeric characters in keys, then you'll have to use [bracket notation](/{{< latest "flux" >}}/data-types/composite/record/#bracket-notation) in Flux.

### Avoid duplicate names for tags and fields

Avoid using the same name for a [tag key](/influxdb/v2.0/reference/glossary/#tag-key) and a [field key](/influxdb/v2.0/reference/glossary/#field-key), which may produce unexpected results when querying data.

{{% cloud-only %}}

{{% note %}}
Use [explicit bucket schemas]() to enforce unique tags and field names within a schema.
{{% /note %}}

{{% /cloud-only %}}


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
