---
RFC: 4
Status: Proposed
---

# ValkeyTimeSeries Module RFC

## Abstract

The proposed feature is ValkeyTimeSeries which is a [Rust](https://www.rust-lang.org/) based Module that brings a native time series data type to Valkey.

To help users migrate from Redis and RedisTimeSeries, as well as capitalize on existing OSS RedisTimeSeries client libraries, 
the module is designed to be API-compatible with Redis Ltd.’s RedisTimeSeries.

## Motivation

Support for a time series data type in Valkey is essential for developers wanting to use Valkey in domains like observability, 
monitoring, IoT, and analytics.

Valkey TimeSeries aims to enable the unique characteristics of time-series data:

* Efficient Storage: optimize for high write throughput and data compression, enabling us to handle the continuous influx of large volumes of timestamped data.
* Real-Time Analytics: support real-time querying and computation, allowing users to detect trends, anomalies, or patterns as they occur.
* Time-centric Queries: allow for time-based aggregations and querying over specific time intervals with high efficiency.

Although Valkey time series modeling can be achieved natively in ValKey using built-in types like `stream`s and `zset`s,
a dedicated time series data type can provide better performance, memory efficiency, and ease of use for time series workloads.

Redis Ltd.‘s TimeSeries module is published under a proprietary license, hence cannot be distributed freely with ValKey.

## Design Considerations

The ValkeyTimeSeries module brings a time series module data type to Valkey and provides commands to create 
time series, operate on them (add sample, query, perform aggregations, compactions, downsampling and joins, etc.).
It allows customization of the properties of each series (compression, retention, compactions, etc.) through commands and 
configurations. 

We aim to be efficient both in memory usage and performance. This is achieved through:
* use of parallelism and background tasks where appropriate (multi-chunk fetches, series compactions, index maintenance, etc.)
* efficient memory management (interning for labels, index prefix compression, Roaring Bitmaps for series ids)
* efficient filtering and indexing

We have the following terminologies:
* TimeSeries Object: The top level structure representing the data type. It contains metadata and a list of lower-level
                chunks containing the actual sample data.
* Sample: A tuple of timestamp and value.
* Chunk: A container for samples. Chunks have configurable size and encoding policies.

### Enhancements over RedisTimeSeries
* Explicit support for multiple databases, allowing users to create time series objects in different databases.
* Joins: ValkeyTimeSeries supports joins between time series objects, including INNER, OUTER, and ASOF joins
* Filtering: support filtering using Prometheus style selectors
* Compaction: support for creating compaction rules based on other compactions.
* Metadata: support for returning metadata on time series objects (label names, label values, cardinality, etc)
* Rounding: support for rounding sample values to specified precision. This is enforced for all samples in a time series.
* Active expiration: support for active pruning of time series data based on retention.
* Developer Ergonomics: support for relative timestamps in queries, e.g. `TS.RANGE key -6hrs -3hrs`, unit suffixes (e.g. `1s`, `3mb`, `20K`), 
    and a more expressive query language.

### Module OnLoad

Upon loading, the module registers a new time series module-based data type, creates time series (TS.*) commands,
timeseries specific configurations, and the TimeSeries ACL category.

* Module name: ts
* Data type name: `vktseries`
* Module shared object file name: valkey_timeseries.so

With the Module name as "ts", ValkeyTimeSeries is compatible with RedisTimeseries in its Module name which is accessible by clients
through HELLO, MODULE LIST, and INFO commands. Also, metrics and configs will be prefixed with this name (by design for Modules).

Regarding the Module Data type name, because ValkeyTimeSeries's Module Data type (the current version) is not compatible with
RedisTimeSeries, it is not named the same as RedisTimeSeries's. 

### Module Unload

Once the Module has been loaded, the `MODULE UNLOAD` will be rejected since Module Data type is created on load.
Valkey does not allow unloading of Modules that export a module data type.

```
127.0.0.1:6379> MODULE UNLOAD timeseries
(error) ERR Error unloading module: the module exports one or more module-side data types, can't unload
```

### Persistence

ValkeyTimeSeries implements persistence-related Module data type callbacks for the TimeSeries data type:

* rdb_save: Serializes timeseries objects to RDB.
* rdb_load: Deserializes time series objects from RDB.
* aof_rewrite: Emits commands into the AOF during the AOF rewriting process.

### RDB Compatibility with RedisTimeSeries

ValkeyTimeSeries is _**not**_ RDB Compatible with RedisTimeSeries.

The metadata that gets written to the RDB is specific to the Module data type's structure and struct members.
Additionally, the data within the underlying time series is specific to the implementation of
the time series.

Because of this, it is not possible to be RDB compatible with RedisTimeSeries.

### AOF Rewrite handling

Module data types (including timeseries) can implement a callback function that will be triggered for TimeSeries objects to rewrite
its data as command/s. From the AOF callback, we will handle AOF rewrite by saving a TS.LOAD command with the key, TTL, and
serialized value of the corresponding time series object.

### Migrating workloads from RedisTimeSeries:

Customers that currently use RedisTimeSeries can move to ValkeyTimeSeries using two approaches:

1. Create the time series objects to have the same properties using TS.CREATE / TS.ALTER on Valkey (with ValkeyTimeSeries loaded).
Re-populate the time series objects by inserting items by moving the existing samples to the Valkey server
(with ValkeyTimeSeries loaded). The workload can be moved without any errors since we are API Compatible.

Pros
* This is the simplest option in terms of effort - assuming the user is fine with recreating time series objects and adding
    items on the Valkey server (with ValkeyTimeSeries).

Cons
* The user will need to re-create the time series (using TS.CREATE/TS.ADD) and populate these objects **AFTER**
    switching to Valkey (with ValkeyTimeSeries).

Users can generate an AOF file from a server (that has the RedisTimeSeries module loaded) and with an ongoing timeseries workload
that creates time series and inserts items into them. Next, this can be re-played on a Valkey Server (with ValkeyTimeSeries loaded).
Then, the user can move their existing workload to the Valkey server (with ValkeyTimeSeries loaded).

### Memory Management

On Module Load, the Rust Module overrides the memory allocator that delegates the allocation and deallocation tasks
to the Valkey server using the existing Module APIs: ValkeyModule_Alloc and ValkeyModule_Free. This API panics if unable
to allocate enough memory.

The timeseries data type also supports memory management related callbacks:

* free: Frees the time series object when it is deleted, expired, or evicted.
* defrag: Supports active defrag for TimeSeries objects
* mem_usage: Reports bytes used by a TimeSeries object and is used by the MEMORY USAGE command
* copy: Supports a deep copy of time series objects and is used by the COPY command
* free_effort: Determine whether the TimeSeries object's memory needs to be lazily reclaimed or synchronously freed.

### Replication
Every TimeSeries based write operation (creation, adding and removing samples) will be replicated to replica nodes.

## Specification

### TimeSeries Structure
Each time series object is represented by a sorted list of chunks, each containing a list of samples. Each sample is a tuple
of 64bit epoch-based timestamp and a 64bit float value. 

A time series is optionally identified by a series of label-value pairs used to retrieve the series in queries. Given that these
labels are used to group semantically similar time series, they are necessarily duplicated. We take advantage of this fact to 
intern label-value pairs, meaning that only a single allocation is made per unique pair, irrespective of the number of series
it occurs in.

The time series object also contains metadata like the retention policy, compaction policy, etc.

### TimeSeries Value Encoding
TimeSeries Chunks have configurable sample encoding strategies, defaulting to Gorilla XOR compression.
Chunks can also be configured to store values as Uncompressed, but this is provided only for compatibility with RedisTimeseries.

### TimeSeries Indexing
In addition to labels, a time series is uniquely identified by an opaque unsigned 64bit int. Each label-value pair
is mapped to the id of each series which contains that attribute. The mapping is implemented as an [Adaptive Radix Tree (ART)](https://db.in.tum.de/~leis/papers/ART.pdf) (pdf), 
where each node is a 64bit [Roaring BitMap](https://roaringbitmap.org/about/).

The ART allows for efficient lookups and insertions, while the Roaring BitMap performs fast set operations on the series ids.
In addition, the ART supports path compression for additional memory savings based on our indexing scheme (see below).

### TimeSeries Indexing Scheme
The ART is used to index time series based on their labels. For each unique combination of label and value, we create a key by concatenating 
the label and value strings. E.g. "region=us-west-2". This key is used to manage a 64bit roaring bitmap that contains the ids of all time series 
that have that label-value pair. To retrieve ids for a given list of label-value pairs, we look up the keys in the ART and perform an intersection.
The ART natively supports range queries, so we can efficiently find keys with a given prefix. For example, we can search on the prefix "region=" to
find all time series with a label "region".

We also maintain a mapping from id to a valkey key to retrieve the time series after querying.

### TimeSeries Filter Enhancements
We support an extension to the RedisTimeseries filter syntax to support regex-based filtering using the operators `=~` and `!~`. 
For example:

* `TS.QUERYINDEX region=~us-west.*` will return all time series that have a label `region` with a value that starts with `us-west`.

* `TS.QUERYINDEX region!~us-west.*` will return all time series that have a label `region` with a value that does not start with `us-west`.

In addition, we add Prometheus style selectors to the filter syntax (essentially an [Instant Vector](https://promlabs.com/blog/2020/07/02/selecting-data-in-promql/#instant-vector-selectors) selector). For example:

* `TS.QUERYINDEX latency{region=~"us-west-*",service="inference"}` will return all series recording latency for the inference service in all us west regions. 

* `TS.MRANGE -6hrs -3hrs {service="inference"}` will return samples recorded between 3 and 6 hours ago for the inference service across all series 

We also support `"OR"` matching for Prometheus style selectors. For example:

* `TS.QUERYINDEX latency{region="us-west" or region="us-east"}` will return all series recording latency samples in either `us-west` or `us-east` regions.

As per Prometheus conventions, all regex matchers are anchored. For example, a matcher of `env=~"foo"` is treated as `env=~"^foo$"`, so any anchors 
used in the query will be redundant. 

Note that for selectors of the form `metric{label="value"}`, `metric` is matched against the reserved `__name__` label. In other words,
the series is expected to have a label `__name__` with the value `metric`.

### RDB Format

During RDB save, the Module data type callback is invoked, and we store series-specific data.


### TimeSeries Command API

The following are supported TimeSeries commands with API syntax compatible with RedisTimeSeries:

### TS.CREATE

#### Syntax
```
TS.CREATE key
  [RETENTION retentionPeriod]
  [ENCODING <COMPRESSED|UNCOMPRESSED>]
  [CHUNK_SIZE chunkSize]
  [DUPLICATE_POLICY policy]
  [DEDUPE_INTERVAL duplicateTimediff]
  [[LABELS [label value ...] | METRIC metricName]
```
Create a new time series. This command creates a new time series with the specified key and optional properties.

#### Options
- **ENCODING**: The encoding to use for the timeseries. Default is `COMPRESSED`.
- **DUPLICATE_POLICY**: The policy to use for duplicate samples. Default is `BLOCK`.

### Required arguments

<summary><code>key</code>
is key name for the time series.
</summary>

### Optional Arguments
<summary><code>retentionPeriod</code>
The period of time for which to keep series samples. Retention can be specified as an integer indication
the duration as milliseconds, or a duration expression like `3wk`
</summary>

<summary><code>chunkSize</code>
The chunk size for the timeseries, in bytes. Default is `4096`.
</summary>

<summary><code>dedupeInterval</code>
Limits sample ingest to the timeseries. If a sample arrives less than `dedupeInterval` from the most
recent sample it is ignored. Default is `0`
</summary>

---
### TS.CREATERULE
#### Syntax

```
TS.CREATERULE sourceKey destKey 
    AGGREGATION aggregator bucketDuration
    [alignTimestamp]
```

Create a compaction rule that aggregates data from `sourceKey` into `destKey` using the specified aggregation
function and bucket duration.

---
### TS.ALTER

#### Syntax
```
TS.ALTER key
  [RETENTION retentionPeriod]
  [CHUNK_SIZE chunkSize]
  [DUPLICATE_POLICY policy]
  [DEDUPE_INTERVAL duplicateTimediff]
  [[LABELS [label value ...] | METRIC metricName]
```

Alter an existing time series. This command allows you to change the properties of a time series, such as its 
retention period, chunk size, and duplicate policy.

#### Options
- **ENCODING**: The encoding to use for the timeseries. Default is `COMPRESSED`.
- **DUPLICATE_POLICY**: The policy to use for duplicate samples. Default is `BLOCK`.

#### Required arguments

<summary><code>key</code>
is key name for the time series.
</summary>

#### Optional Arguments
<summary><code>retentionPeriod</code>
The period of time for which to keep series samples. Retention can be specified as an integer indication
the duration as milliseconds, or a duration expression like `3wk`
</summary>

<summary><code>chunkSize</code>
The chunk size for the timeseries, in bytes. Default is `4096`.
</summary>

---
### TS.ADD

#### Syntax
```
TS.ADD key timestamp value
```

Add a sample to a time series. If the key does not exist, it is created with the specified timestamp and value.


#### Required Arguments

<summary><code>key</code>
is key name for the time series.
</summary>

<summary><code>timestamp</code>
The timestamp of the incoming sample
</summary>

<summary><code>value</code>
the value of the sample (double)
</summary>

#### Return
The timestamp of the added sample.

---
### TS.DEL

#### Syntax

```
TS.DEL key fromTimestamp toTimestamp
```

Deletes data for a selection of series in a time range.

#### Options

- **key**: the key being deleted from.
- **fromTimestamp**: Start timestamp, inclusive. Optional.
- **toTimestamp**: End timestamp, inclusive. Optional.

#### Return

- the number of samples deleted.

#### Error

Return an error reply in the following cases:

TODO

#### Examples

```
TS.DEL requests:status:200 587396550 1587396550
```
---
### TS.DELETERULE
#### Syntax

```
TS.DELETERULE sourceKey destKey
```
Delete a compaction rule that aggregates data from `sourceKey` into `destKey`.

---

### TS.MADD

#### Syntax
```
TS.MADD <key> <item> [<item> ...]
```
see https://redis.io/docs/latest/commands/ts.madd/
When compactions are supported, we should use `rust's` parallelism to perform the compactions in the background.

---
### TS.GET

```
TS.GET <key> [LATEST]
```

see https://redis.io/docs/latest/commands/ts.get/

Get the last sample from a series.

---
### TS.MGET

```
TS.MGET 
    [LATEST] 
    [WITHLABELS | <SELECTED_LABELS label...>] 
    FILTER filterExpr...
```
Get the last sample from a multiple series specified by a filter.
see https://redis.io/docs/latest/commands/ts.mget/

---

### TS.INCRBY

#### Syntax
```
TS.INCRBY key delta
  [TIMESTAMP timestamp]
  [RETENTION retentionPeriod]
  [ENCODING <COMPRESSED|UNCOMPRESSED>]
  [CHUNK_SIZE chunkSize]
  [DUPLICATE_POLICY policy]
  [DEDUPE_INTERVAL duplicateTimediff]
  [[LABELS [label value ...] | METRIC metricName]
```
increment the value of the last sample in a time series

---
### TS.DECRBY

#### Syntax
```
TS.DECRBY key delta
  [TIMESTAMP timestamp]
  [RETENTION retentionPeriod]
  [ENCODING <COMPRESSED|UNCOMPRESSED>]
  [CHUNK_SIZE chunkSize]
  [DUPLICATE_POLICY policy]
  [DEDUPE_INTERVAL duplicateTimediff]
  [[LABELS [label value ...] | METRIC metricName]
```

Increment the value of the last sample in a time series
see https://redis.io/docs/latest/commands/ts.decrby/

---
### TS.INFO

#### Syntax

```
TS.INFO <key>
```

Get information about a time series.

see https://redis.io/docs/latest/commands/ts.info/

---
### TS.RANGE

#### Syntax
```
TS.RANGE key fromTimestamp toTimestamp 
    [LATEST]
    [FILTER_BY_TS ts...] 
    [FILTER_BY_VALUE min max]
    [COUNT count] 
    [AGGREGATION aggregator bucketDuration [ALIGN align] [BUCKETTIMESTAMP timestamp] [EMPTY]]
```

Query a range of data

see https://redis.io/docs/latest/commands/ts.range/

---
### TS.REVRANGE

#### Syntax

```
TS.REVRANGE key fromTimestamp toTimestamp 
    [LATEST]
    [FILTER_BY_TS ts...] 
    [FILTER_BY_VALUE min max]
    [COUNT count] 
    [AGGREGATION aggregator bucketDuration [ALIGN align] [BUCKETTIMESTAMP timestamp] [EMPTY]]
```

Query a range of data in reverse order

see https://redis.io/docs/latest/commands/ts.revrange/

---
### TS.MRANGE

#### Syntax

```
TS.MRANGE fromTimestamp toTimestamp
    [LATEST
    [FILTER_BY_TS ts...] 
    [FILTER_BY_VALUE min max]
    [COUNT count] 
    [REDUCE operator]
    [GROUPBY label REDUCE reducer]
    [AGGREGATION aggregator bucketDuration [ALIGN align] [BUCKETTIMESTAMP timestamp] [EMPTY]]
    [WITHLABELS | <SELECTED_LABELS label...>]
```

Query a range of data across multiple series

see https://redis.io/docs/latest/commands/ts.mrange/

---
### TS.MREVRANGE

#### Syntax

```
TS.MREVRANGE fromTimestamp toTimestamp 
    [FILTER_BY_TS ts...] 
    [FILTER_BY_VALUE min max]
    [COUNT count] 
    [REDUCE operator]
    [GROUPBY label REDUCE reducer]
    [AGGREGATION aggregator bucketDuration [ALIGN align] [BUCKETTIMESTAMP timestamp] [EMPTY]]
    [WITHLABELS | <SELECTED_LABELS label...>]
```

Query a range of data across multiple series in the reverse order

see https://redis.io/docs/latest/commands/ts.mrevrange/

---
### TS.QUERYINDEX filterExpression

#### Syntax

```
TS.QUERYINDEX filterExpression
```

Returns the list of time series keys that match the filter expression(s).

see https://redis.io/docs/latest/commands/ts.mrange/

---
### New Commands

The following are NEW commands that are not included in RedisTimeSeries:

---
### TS.CARD

#### Syntax

```
TS.CARD FILTER filter... [START fromTimestamp] [END toTimestamp]
```

Returns the number of unique time series that match a given filter set. A time range can optionally be provided to
restrict the results to only series which have samples in the range `[fromTimestamp, toTimestamp]`.

#### Required arguments

<code>filter</code>

Repeated series selector argument that selects the series to return. At least one filter argument must be provided.

#### Optional Arguments
<code>fromTimestamp</code>

Start timestamp, inclusive. Results will only be returned for series which have samples in the range `[fromTimestamp, toTimestamp]`

<code>toTimestamp</code>

End timestamp, inclusive.


#### Return

[Integer number](https://redis.io/docs/reference/protocol-spec#resp-integers) of unique time series.

---
### TS.LABELNAMES

#### Syntax

```
TS.LABELNAMES FILTER selector... [START fromTimestamp] [END toTimestamp]
```

Returns a list of label names for select series. If a time range is specified, only labels from series which have data in the date range [`fromTimestamp` .. `toTimestamp`] are returned.

### Required Arguments
<code>fromTimestamp</code>
Repeated series selector argument that selects the series to return. At least one selector argument must be provided..


### Optional Arguments
<code>fromTimestamp</code>

If specified along with `toTimestamp`, this limits the result to only labels from series which
have data in the date range [`fromTimestamp` .. `toTimestamp`]

<code>toTimestamp</code>

If specified along with `fromTimestamp`, this limits the result to only labels from series which
have data in the date range [`fromTimestamp` .. `toTimestamp`]

#### Return

An array of string label names.

**`TS.LABELNAMES FILTER up process_start_time_seconds{job="prometheus"}`**
```
1) "__name__",
2) "instance",
3) "job"

```

---
### TS.LABELVALUES

#### Syntax

```
TS.LABELVALUES label [START fromTimestamp] [END toTimestamp]
```

Returns a list of label values for a provided label name. Optionally a time range can be specified to limit the result to only 
labels from series which have data in the date range [`fromTimestamp` .. `toTimestamp`].

#### Required Arguments

<code>label</code>

The label name for which to retrieve mut values.


#### Optional Arguments

<code>fromTimestamp</code>

If specified along with `toTimestamp`, this limits the result to only labels from series which
have data in the date range [`fromTimestamp` .. `toTimestamp`]


<code>toTimestamp</code>

If specified along with `fromTimestamp`, this limits the result to only labels from series which
have data in the date range [`fromTimestamp` .. `toTimestamp`]


#### Return

An array of string values.

#### Error

Return an error reply in the following cases:

- Invalid options.
- TODO.

#### Examples

This example queries for all label values for the "`region`" label:
```
TS.LABELVALUES region
```
```
1) "us-east-1",
2) "us-east-2"
1) "us-west-1",
2) "us-west-2"
```

---
### TS.STATS

#### Syntax

```
TS.STATS [limit limit]
```

Returns various cardinality statistics about the time series data.

`limit` - the number of returned items to a given number for each set of statistics. By default, 10 items are returned.

#### Example
```
TS.STATS LIMIT 5
```

```
 1) totalSeries
 2) (integer) 508
 3) seriesCountByMetricName
    1) 1) "net_conntrack_dialer_conn_failed_total"
       2) (integer) 20
    2) 1) "prometheus_http_request_duration_seconds_bucket"      
       2) (integer) 20
    3) 1) "http_request_latency_seconds_count"
       2) (integer) 20
 4) labelValueCountByLabelName
    1) 1) "__name__"
       2) (integer) 211
    2) 1) "event"      
       2) (integer) 4
 5) memoryInBytesByLabelName
    1) 1) "__name__"
       2) (integer) 8266
    2) 1) "instance"      
       2) (integer) 28
 6) seriesCountByLabelValuePair
    1) 1) "job=prometheus"
       2) (integer) 430
    2) 1) "instance=localhost:9090"      
       2) (integer) 425 
```

---
### TS.JOIN

#### Syntax

```
TS.JOIN leftKey rightKey fromTimestamp toTimestamp
    [[INNER] | [FULL] | [LEFT] | [RIGHT] | [ANTI] | [SEMI] | [ASOF [PREVIOUS | NEXT | NEAREST] [tolerance] ALLOW_EXACT_MATCH]]
    [FILTER_BY_TS ts...]
    [FILTER_BY_VALUE min max]
    [COUNT count]
    [REDUCE operator]
    [AGGREGATION aggregator bucketDuration [ALIGN align] [BUCKETTIMESTAMP timestamp] [EMPTY]]
```

Join two time series on sample timestamps. Performs an INNER join by default.

#### Required arguments

<summary><code>leftKey</code>
is the key name for the time series being joined.
</summary>

<summary><code>rightKey</code>
is the key name for the right time series.
</summary>

Both keys must have been created before `TS.JOIN` is called.

<summary><code>fromTimestamp</code>
`fromTimestamp` is the first timestamp or relative delta from the current time of the request range.
</summary>

<summary><code>toTimestamp</code>
`toTimestamp` is the last timestamp of the requested range, or a relative delta from `fromTimestamp`
</summary>


#### Optional arguments

<summary><code>LEFT</code>
A `LEFT` join outputs the matching samples between both tables. In case no samples match from the left series, it returns 
those items with null values.

</summary>

<summary><code>RIGHT</code>
A `RIGHT` join outputs all samples in the right series. In case no samples match from the left series, it returns
those items with null values.

</summary>

<summary><code>INNER</code>
Specifies an `INNER` join. A row is generated for samples with matching timestamps in the selected range.
</summary>

<summary><code>ANTI</code>
An `ANTI` join returns samples for which no matching timestamp exists in the `right` series.

</summary>

<summary><code>SEMI</code>
An `SEMI` join returns samples for which no corresponding timestamp exists in the `right` series. It does not return any 
values from the right table.

The main purpose is to filter the left series based on the existence of related records in the right table, not to
combine data from both tables.
</summary>

<summary><code>FULL</code>
Specifies an FULL join. Returns samples from both left and right series. If no matching rows exist for the row in the left
series, the value of the right series will have nulls. Correspondingly, the value of the left series will have nulls if
there are no matching rows for the sample in the right series.
</summary>

<summary><code>ASOF [PREVIOUS | NEXT | NEAREST] tolerance [ALLOW_EXACT_MATCH [true|false]]</code>

`ASOF` joins match each sample in the left series with the closest preceding or following sample in the right series based on
timestamps. They are particularly useful for analyzing time-series data where records from different sources may not have
perfectly aligned timestamps. ASOF joins solve the problem of finding the value of a varying property at a specific point in time.

#### How It Works
For each sample in the left table, the join finds the closest matching value from the right table.
- `PREVIOUS` selects the last row in the right series whose timeseries is less than or equal to the left’s timestamp.
- `NEXT` (default) selects the first row in the right series whose timestamp is greater than or equal to the left’s timestamp.
- `NEAREST` selects the last row in the right series whose timestamp is nearest to the left’s timestamp.

`tolerance` sets a limit on how far apart the timestamps can be while still considering them a match.
The tolerance can be specified as:
- An integer representing milliseconds
- A duration specified as a string, e.g., 2m

`ALLOW_EXACT_MATCH` is a boolean flag that determines whether to allow exact matches between the left and right series.


If not specified, there is no tolerance limit (equivalent to an infinite tolerance). When set, JOIN ASOF will only match
keys within the specified tolerance range. Any potential matches outside this range will be treated as no match.

The tolerance works in conjunction with the 'direction' parameter.
- For example, with strategy = `PREVIOUS` (the default), it looks for the nearest timestamp within the tolerance range that is less
  than or equal to the timestamp of the left sample.


#### Example
Suppose we want to get the spreads between buy and sell trades in a trading application

```
TS.JOIN trades:buy trades:sell -1hr * ASOF NEAREST 2ms REDUCE sub
```

The result has all samples from the `buy` series joined with samples from the `sell` series. For each timestamp from the
`buy` series, the query looks for a timestamp that nearest to it from the `sell` series, within a tolerance of
2 milliseconds. If no matching timestamp is found, NULL is inserted.

The `sub` transform function is then supplied to subtract the `sell` value from the `buy` value for each sample returned.

The tolerance parameter is particularly useful when working with time series data where exact matches are rare, but you
want to find the closest match within a reasonable time frame.

It helps prevent incorrect matches that might occur if the nearest available data point is too far away in time or value.

</summary>

<summary><code>COUNT count</code>
the maximum number of samples to return.
TODO: if used with aggregation, this specifies the number of returned buckets as opposed to the number of samples
</summary>

<summary><code>operator</code>

performs an operation on the value in each returned row.

`operator` takes one of the following types:

| `operator`   | Description                                                             |
|--------------|-------------------------------------------------------------------------| 
| `abs_diff`   | abs(`left` - `right`)                                                   |
| `and`        | Returns `left` if either value is NAN, `right` otherwise                |
| `avg`        | Arithmetic mean of both mut values                                      |
| `default`    | If `left` is is NaN, return `right`, else `left`                        | 
| `div`        | `left` / `right`                                                        |
| `eq`         | Returns 1 if `left` == `right`, 0 otherwise                             |
| `gt`         | Returns 1 if `left` > `right`, otherwise returns 0                      |
| `gte`        | Returns 1 if left is greater than or equals right, otherwise returns 0  |
| `if`         | Returns `left` if `right` is not NaN. Otherwise, NaN is returned.       |
| `ifnot`      | returns `left` if `right` is NaN. Otherwise, NaN is returned.           |
| `lt`         | Returns 1 if `left` > `right`, otherwise returns 0                      |
| `lte`        | Returns 1 if `left` is less than or equals `right`, otherwise returns 0 |
| `min`        | Minimum value                                                           |
| `max`        | Maximum value                                                           | 
| `mul`        | `left` * `right`                                                        |
| `ne`         | Returns 1 if `left` equals `right`, otherwise returns 0                 |
| `pct_change` | The percentage change of `right` over `left`                            |
| `pow`        | `left` ^ `right`                                                        |
| `sgn_diff`   | sgn(`left` - `right`)                                                   |
| `sub`        | `left` - `right`                                                        |
| `sum`        | `left` + `right`                                                        |
| `or`         | return the first non-NaN item. If both are NaN, it returns NaN.         |
| `unless`     | Returns Null unless `left` equals `right`                               |

</summary>

#### Return value

Returns one of these replies:

- @simple-string-reply - `OK` if executed correctly
- @error-reply on error (invalid arguments, wrong key type, etc.), when `sourceKey` does not exist, when `destKey` does not exist, when `sourceKey` is already a destination of a compaction rule, when `destKey` is already a source or a destination of a compaction rule, or when `sourceKey` and `destKey` are identical

#### Examples


Create a time series to store the temperatures measured in Mexico City and Toronto.

```
127.0.0.1:6379> TS.CREATE temp:CDMX temp{location="CDMX"}
OK
127.0.0.1:6379> TS.CREATE temp:TOR temp{location="TOR"}
OK
```

... Add data


Next, run the join.

```
127.0.0.1:6379> TS.JOIN temp:CDMX temp:TOR REDUCE min 
```


### Possible Future Enhancements

* Work is in progress to support [PromQL](https://prometheus.io/docs/prometheus/latest/querying/basics/) as a query language, 
including transform, aggregation and rollup function support. A stretch goal is to support alerts and notifications based on the query results.

* We may support a tiered storage model where data is moved to higher compression chunks (with higher access latency) after a certain period of time. 

* Use the [augurs](https://github.com/grafana/augurs) library to support more complex queries, like forecasting and anomaly detection.


### Configurations

The default properties for TimeSeries can be controlled using configs. The values of the
configs below are only used on a timeseries if the user does not specify the properties explicitly. Example: Using
TS.CREATE or TS.ADD can override the default properties.

Supported Module configurations:
1. _ts-retention-policy_: The default retention policy for time series (ms). Default to 0, which means no retention policy.
2. _ts-duplicate-policy_: The default duplicate policy for time series. Default to "LAST", which means the last sample is kept when duplicates are added.
3. _ts-chunk-size-bytes_: Controls the default chunk memory capacity. When create operations (Ts.CREATE/TS.ADD/TS.INCRBY/TS.DECRBY) are used, the timeseries created
    will use the capacity specified by this config. 
4. _ts-encoding_: Controls the default chunk encoding. Default to "COMPRESSED", which is the Gorilla XOR compression.
5. _ts-ignore-max-time-diff_: Defines the max delta between timestamps to consider them a duplicate. Default to false.
6. _ts-ignore-max-val-diff_: The maximum delta between values to consider them a duplicate.
7. _ts-round-digits_: Controls the default number of digits to round input sample values to. Default to 255.
8. _ts-significant_digits_: Controls the default number of significant digits to round input sample values to. Default to 255.


### ACL

The ValkeyTimeSeries module will introduce a new ACL category - `@timeseries`.

There are four existing ACL categories that are updated to include new TimeSeries commands: `@read`, `@write`, `@fast`, `@slow`.

### Keyspace Event Notification

Every timeseries-based write command (that involves mutation as explained in the section above) will be made to publish
a keyspace event after the data is mutated. Commands include: TS.CREATE, TS.ADD, TS.MADD.

* Event type: VALKEYMODULE_NOTIFY_GENERIC
* Event name: One of the two event names will be published based on the command & scenario:
 * ts.add
 * ts.create
 * ts.alter
 * ts.madd
 * ts.del


Users can subscribe to the time series events via the standard keyspace event pub/sub. For example,

```text
1. enable keyspace event notifications:
    valkey-cli config set notify-keyspace-events KEA
2. subsbcribe to keyspace & keyevent event channels:
    valkey-cli psubscribe '__key*__:*'
```

## References
* [valkey-timeseries GitHub Repo](https://github.com/ccollie/valkey-timeseries)
* [RedisTimeSeries](https://oss.redislabs.com/redistimeseries/)
* [Prometheus](https://prometheus.io/)
* [Adaptive Radix Tree (ART)](https://db.in.tum.de/~leis/papers/ART.pdf)
* [Roaring Bitmaps](https://roaringbitmap.org/about/)
