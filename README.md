# Elasticsearch Drift Plugin  [![CircleCI](https://circleci.com/gh/OpenNMS/elasticsearch-drift-plugin.svg?style=svg)](https://circleci.com/gh/OpenNMS/elasticsearch-drift-plugin)

Time series aggregation for flow records.

|   Drift Plugin  | Elasticsearch     | Release date   |
|-----------------|-------------------|:--------------:|
| 1.0.x           | 6.2.4             |  May 16th 2018 |
| 1.1.0           | 6.5.4             |  Feb 2019      |
| 1.2.0           | 6.7.0             |  Apr 2019      |

## Overview

This plugin provides a new aggregation function `proportional_sum` that can be used to:

1. Group documents that contain a date range into multiple buckets
1. Calculate a sum on a per bucket basis using a ratio that is proportional to the range of time in which the document spent in that bucket.

This aggregation function behaves like a hybrid of both the `Metrics` and `Bucket` type aggregations since we both create buckets and calculate a new metric.

## Installation

### RPM

Install the package repository:
```
sudo yum install https://yum.opennms.org/repofiles/opennms-repo-stable-rhel7.noarch.rpm
sudo rpm --import https://yum.opennms.org/OPENNMS-GPG-KEY
```

Install the package:
```
sudo yum install elasticsearch-drift-plugin
```

### Debian

Create a new apt source file (eg: `/etc/apt/sources.list.d/opennms.list`), and add the following 2 lines:
```
deb https://debian.opennms.org stable main
deb-src https://debian.opennms.org stable main
```

Import the packages' authentication key with the following command:
```
wget -O - https://debian.opennms.org/OPENNMS-GPG-KEY | sudo apt-key add -
```

Install the package:
```
sudo apt-get update
sudo apt-get install elasticsearch-drift-plugin
```

## Use Case

We are interested in generating time series for Netflow records stored in Elasticsearch.
Each Netflow record is stored as a separate document and contains the following fields of interest:

```json
{
  "timestamp": 460,
  "netflow.first_switched": 100,
  "netflow.last_switched": 450,
  "netflow.bytes": 350
}
```

For this record, we’d like to be able to generate a time series with start=0, end=500, step=100, and have the following data points:

```
t=0, bytes=0
t=100, bytes=100
t=200, bytes=100
t=300, bytes=100
t=400, bytes=50
t=500, bytes=0
```

In this case, each step (or bucket) would contain a fraction of the bytes, relative to how much of the flow falls into that step.
We assume that the flow bytes are evenly spread across the range and if were multiple flow records in a single step we would sum of the corresponding bytes.

Since the existing aggregation facilities in Elasticsearch don't support this behavior, we've gone ahead and developed our own.

## Usage

Using the record above, the `proportional_sum` aggregation can be used as follows:

### Request

```json
{
  "size": 0,
  "aggs": {
    "bytes_over_time": {
      "proportional_sum": {
        "fields": [
          "netflow.first_switched",
          "netflow.last_switched",
          "netflow.bytes"
        ],
        "interval": 100,
        "start": 0,
        "end": 500
      }
    },
    "bytes_total": {
      "sum": {
        "field": "netflow.bytes"
      }
    }
  }
}
```

The `fields` options must be present, and must reference the following document fields in order:

1. The start of the range
1. The end of the range
3. The value

The `interval` can be set a string with a date format, or a numeric value representing the number of milliseconds between steps.

The `start` and `end` fields are optional and take a unix timestamp in milliseconds.
When set, the generated buckets will be limited to ones that fall within this range.
This allows for the documents themselves to be contain wider ranges for which we do not want generate buckets/series for.

### Response

```json
{
  "took" : 2,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : 1,
    "max_score" : 0.0,
    "hits" : [ ]
  },
  "aggregations" : {
    "bytes_total" : {
      "value" : 350.0
    },
    "bytes_over_time" : {
      "buckets" : [
        {
          "key" : 100,
          "doc_count" : 1,
          "value" : 100.0
        },
        {
          "key" : 200,
          "doc_count" : 1,
          "value" : 100.0
        },
        {
          "key" : 300,
          "doc_count" : 1,
          "value" : 100.0
        },
        {
          "key" : 400,
          "doc_count" : 1,
          "value" : 50.0
        }
      ]
    }
  }
}
```

Here we can see that many buckets were generated for the single document and that the value was spread into these buckets accordingly.

## Building and installing from source

To compile the plugin run:
```
mvn clean package
```

Next, ensure setup an Elasticsearch instance using the same version that is defined in the `pom.xml`.
The version must match exactly, otherwise Elasticsearch will refuse to start.

Install the plugin using:
```
/usr/share/elasticsearch/bin/elasticsearch-plugin install file:///path/to/elasticsearch-drift/plugin/target/releases/elasticsearch-drift-plugin-1.0.0-SNAPSHOT.zip
```

