---
title: Troubleshoot issues writing data
seotitle: Troubleshoot issues writing data
list_title: Troubleshoot issues writing data
weight: 106 >
  Troubleshoot issues writing data. Find response codes for failed writes. Discover how writes fail, from exceeding rate or payload limits, to syntax errors and schema conflicts. Find suggestions to fix failures.
menu:
  influxdb_2_0:
    name: Troubleshoot issues
    parent: Write data
influxdb/v2.0/tags: [write, line protocol, errors]
related:
  - /influxdb/v2.0/api/#tag/Write, InfluxDB API /write endpoint
  - /influxdb/v2.0/reference/internals
  - /influxdb/v2.0/reference/cli/influx/write
---
Learn how to handle and recover from errors when writing to InfluxDB.

- [Discover common failure scenarios](#discover-common-failure-scenarios)
- [Review HTTP status codes](#review-http-status-codes)
- [Troubleshoot failures](#troubleshoot-failures)

## Discover common failure scenarios

Write requests made to InfluxDB may fail for a number of reasons.
Common failure scenarios that return an HTTP `4xx` or `5xx` error status code include the following:

- API token was invalid.
- Client or server reached a timeout threshold.
- Data was not formatted correctly.
- Size of the data payload was too large.

Writes may fail partially or completely even though InfluxDB returns an HTTP `2xx` status code for a valid request.
For example, a partial write may occur when InfluxDB writes all points that conform to the bucket schema, but rejects points that have the wrong data type in a field.

## Review HTTP status codes

InfluxDB uses conventional HTTP status codes to indicate the success or failure of a request.
Write requests return the following status codes:

- `204` **Success**: InfluxDB validated the request data format and accepted the data for writing to the bucket.
    {{% note %}}
  `204` doesn't indicate a successful write operation since writes are asynchronous.
  If some of your data did not write to the bucket, see how to [check for rejected points](#review-rejected-points).
    {{% /note %}}

- `400` **Bad request**: The [line protocol](/influxdb/v2.0/reference/syntax/line-protocol/) data in the request is malformed.
  The response body contains the first malformed line in the data. All request data was rejected and not written.
- `401` **Unauthorized**: May indicate one of the following:
  - [`Authorization: Token` header](/influxdb/v2.0/api-guide/api_intro/#authentication) is missing or malformed.
  - [API token](/influxdb/v2.0/api-guide/api_intro/#authentication) value is missing from the header.
  - API token does not have sufficient permissions to write to the organization and the bucket.
    For more information about token types and permissions, see [Manage API tokens](/influxdb/v2.0/security/tokens/).
- `404` **Not found**: A requested resource (e.g. an organization or bucket) was not found.
  The response body contains the requested resource type, e.g. "organization", and resource name.
- `413` **Request entity too large**: All request data is rejected and not written.
  InfluxDB OSS only returns this error if the [Go (golang) `ioutil.ReadAll()`](https://pkg.go.dev/io/ioutil#ReadAll) function raises an error. 
- `422` **Unprocessible entity**: Request data is invalid. `code` and `message` in the response body provide details about the problem.
- `500` **Internal server error**: Default HTTP status for an error.
- `503` **Service unavailable**: Server is temporarily unavailable to accept writes.
  The `Retry-After` header describes when to try the write again.

The `message` property of the response body may contain additional details about the error.

## Troubleshoot failures

If you notice data is missing in your bucket, do the following:

- Check the `message` property in the response body for details about the error
- Verify all lines contain valid syntax, e.g. [line protocol](/influxdb/v2.0/reference/syntax/line-protocol/) or [CSV](/influxdb/v2.0/reference/syntax/annotated-csv/)
- Verify the data types match other data points with the same series.
  For example, did you attempt to write `string` data to an `int` field?
- Verify the timestamps match the [precision parameter](/influxdb/v2.0/write-data/#timestamp-precision).
- Minimize payload size and network errors by [optimizing writes](/influxdb/v2.0/write-data/best-practices/optimize-writes/) 

### Review rejected points

InfluxDB may reject points even if the HTTP request returns "Success".
To get a log of rejected data points, query the [`rejected_points` measurement](/influxdb/v2.0/reference/internals/system-buckets/#_monitoring-bucket-schema) in your organization's `_monitoring` bucket.

```js
from(bucket: "_monitoring")
  |> range(start:-1h)
  |> filter(fn:(r) =>
    r._measurement == "rejected_points"
  )
```

InfluxDB returns `rejected_points` log entries.
Each entry contains the `bucket` where the rejection occurred and a `reason` for the rejection.

```sh
rejected_points,bucket=YOUR_BUCKET_ID,field=YOUR_FIELD,gotType=Float,
  measurement=YOUR_MEASUREMENT,reason=type\ conflict\ in\ batch\ write,
  wantType=Integer count=1i 1627906197091972750
```

#### Troubleshoot field type conflicts

Field type conflicts occur if you attempt to write different data types to a field.
For example, you attempt to write `string` data to an `int` field.
If InfluxDB rejects a point due to a [field type](/influxdb/v2.0/reference/key-concepts/data-elements/#field-value) conflict, the rejected point log entry contains the following data elements:

| Name          | Value                                                                                                                                        |
|:------        |:-----                                                                                                                                        |
| `bucket`      | ID of the bucket that rejected the point.                                                                                                    |
| `measurement` | Measurement name of the point.                                                                                                               |
| `field`       | Name of the field that caused the rejection.                                                                                                 |
| `reason`      | Brief description of the problem. See [reasons for rejected points](#reasons-for-rejected-points)                                            |
| `gotType`     | Received [field](/influxdb/v2.0/reference/key-concepts/data-elements/#field-value) type: `Boolean`, `Float`, `Integer`, or `UnsignedInteger` |
| `wantType`    | Expected [field](/influxdb/v2.0/reference/key-concepts/data-elements/#field-value) type: `Boolean`, `Float`, `Integer`, or `UnsignedInteger` |
| `error`       | Additional error detail.                                                                                                                     |
| `count`       | `1`                                                                                                                                          |
| `<timestamp>` | Time the rejected point was logged.                                                                                                          |

##### Reasons for rejected points

Field type conflicts occur for the following reasons:

| Reason                             | Meaning                                                                                                       |
|:------                             |:-------                                                                                                       |
| `type conflict in batch write`     | The **batch** contains another point with the same series, but one of the fields has a different value type.  |
| `type conflict with existing data` | The **bucket** contains another point with the same series, but one of the fields has a different value type. |

