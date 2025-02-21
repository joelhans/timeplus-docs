#  Query Syntax

## Streaming SQL Syntax{#select-syntax}

Timeplus introduces several SQL extensions to support streaming processing. The overall syntax looks like this:

```sql
SELECT <expr, columns, aggr>
FROM <streaming_window_function>(<table_name>, [<time_column>], [<window_size>], ...)
[WHERE clause]
[GROUP BY clause]
EMIT <window_emit_policy>
SETTINGS <key1>=<value1>, <key2>=<value2>, ...
```

Overall, a streaming query in Timeplus establishes a long HTTP/TCP connection to clients and continuously evaluates the query. It will continue to return results according to the `EMIT` policy until the query is stopped, by the user, end client, or exceptional event. 

Timeplus supports some advanced `SETTINGS` to fine tune the streaming query processing behaviors, listed below:

1. `enable_backfill_from_historical_store=1`. By default, if it's omitted, it's `0`
   * When it's 0, the query engine either loads data from streaming storage, or from historical storage.
   * When it's 1, the query engine evaluates whether it's necessary to load data from historical storage(such as the time range is outside of the streaming storage), or it'll be more efficient to get data from historical storage(for example, count/min/max is pre-computed in historical storage, faster than scanning data in streaming storage).
2. `query_mode=<table|streaming>`. By default, if it's omitted, it's `streaming`. A general setting which decides if the overall query is streaming data processing or historical data processing. 
3. `seek_to=<timestamp|earliest|latest>`. By default, if it's omitted, it's `latest`. A setting which tells Timeplus to seek old data in the streaming storage by timestamp. It can be a relative timestamp or an absolute timestamp. By default, it is `latest`, which tells Timeplus to not seek old data. Example:`seek_to='2022-01-12 06:00:00.000'`, ` seek_to='-2h'`, or ` seek_to='earliest'` 

:::info

Please note, as of Jan 2023, we no longer recommend you use `SETTINGS seek_to=..`. Please use `WHERE _tp_time>='2023-01-01'` or similar. `_tp_time` is the special timestamp column in each raw stream to represent the event time. You can use `>`, `<`, `BETWEEN .. AND` operations to filter the data in Timeplus stream storage. The only exception is [External Stream](external-stream). If you need to scan all existing data in the Kafka topic, you have to run the SQL with seek_to, e.g. `select raw from my_ext_stream settings seek_to='earliest'`

:::

### Streaming Tailing

```sql
SELECT <expr>, <columns>
FROM <table_name>
[WHERE clause]
```

Examples

```sql
SELECT device, cpu_usage
FROM devices_utils
WHERE cpu_usage >= 99
```

The above example continuously evaluates the filter expression on the new events in the stream `device_utils` to filter out events
which have `cpu_usage` less than 99. The final events will be streamed to clients.

### Global Streaming Aggregation {#global}

In Timeplus, we define global aggregation as an aggregation query without using streaming windows like tumble, hop. Unlike streaming window aggregation, global streaming aggregation doesn't slice
the unbound streaming data into windows according to timestamp, instead it processes the unbounded streaming data as one huge big global window. Due to this property, Timeplus for now can't
recycle in-memory aggregation states / results according to timestamp for global aggregation.

```sql
SELECT <column_name1>, <column_name2>, <aggr_function>
FROM <table_name>
[WHERE clause]
EMIT PERIODIC [<n><UNIT>]
```

`PERIODIC <n><UNIT>` tells Timeplus to emit the aggregation periodically. `UNIT` can be ms(millisecond), s(second), m(minute),h(hour),d(day).`<n>` shall be an integer greater than 0.

Examples

```sql
SELECT device, count(*)
FROM device_utils
WHERE cpu_usage > 99
EMIT PERIODIC 5s
```

Like in [Streaming Tail](#streaming-tailing), Timeplus continuously monitors new events in the stream `device_utils`, does the filtering and then
continuously does **incremental** count aggregation. Whenever the specified delay interval is up, project the current aggregation result
to clients.


### Tumble Streaming Window Aggregation {#tumble}

Tumble slices the unbounded data into different windows according to its parameters. Internally, Timeplus observes the data streaming and automatically decides when to
close a sliced window and emit the final results for that window.

```sql
SELECT <column_name1>, <column_name2>, <aggr_function>
FROM tumble(<table_name>, [<timestamp_column>], <tumble_window_size>, [<time_zone>])
[WHERE clause]
GROUP BY [window_start | window_end], ...
EMIT <window_emit_policy>
SETTINGS <key1>=<value1>, <key2>=<value2>, ...
```

Tumble window means a fixed non-overlapped time window. Here is one example for a 5 seconds tumble window:

```
["2020-01-01 00:00:00", "2020-01-01 00:00:05]
["2020-01-01 00:00:05", "2020-01-01 00:00:10]
["2020-01-01 00:00:10", "2020-01-01 00:00:15]
...
```

`tumble` window in Timeplus is left closed and right open `[)` meaning it includes all events which have timestamps
**greater or equal** to the **lower bound** of the window, but **less** than the **upper bound** of the window.

`tumble` in the above SQL spec is a table function whose core responsibility is assigning tumble window to each event in
a streaming way. The `tumble` table function will generate 2 new columns: `window_start, window_end` which correspond to the low and high
bounds of a tumble window.

`tumble` table function accepts 4 parameters: `<timestamp_column>` and `<time-zone>` are optional, the others are mandatory.

When the `<timestamp_column>` parameter is omitted from the query, the table's default event timestamp column which is `_tp_time` will be used.

When the `<time_zone>` parameter is omitted the system's default timezone will be used. `<time_zone>` is a string type parameter, for example `UTC`.

`<tumble_window_size>` is an interval parameter: `<n><UNIT>` where `<UNIT>` supports `s`, `m`, `h`, `d`, `w`.
It doesn't yet support `M`, `q`, `y`. For example, `tumble(my_table, 5s)`.

Timeplus supports 2 emit policies for tumble window, so `<window_emit_policy>` can be:

1. `AFTER WATERMARK`: aggregation results will be emitted and pushed to clients right after a watermark is observed. This is the default behavior when this clause is omitted.
2. `AFTER WATERMARK AND DELAY <interval>`: aggregation results will be held after the watermark is observed until the specified delay reaches. Users can use interval shortcuts for the delay. For example, `DELAY 5s`.

**Note** `watermark` is an internal timestamp which is observed and calculated and emitted by Timeplus and is used to indicate when a streaming window shall close. It is guaranteed to be increased monotonically per stream query.

Examples

```sql
SELECT device, max(cpu_usage)
FROM tumble(device_utils, 5s)
GROUP BY device, window_end
```

The above example SQL continuously aggregates max cpu usage per device per tumble window for table `devices_utils`. Every time a window
is closed, Timeplus emits the aggregation results.

```sql
SELECT device, max(cpu_usage)
FROM tumble(device_utils, 5s)
GROUP BY device, widnow_end
EMIT AFTER WATERMARK DELAY 2s;
```

The above example SQL continuously aggregates max cpu usage per device per tumble window for table `device_utils`. Every time a window
is closed, Timeplus waits for another 2 seconds and then emits the aggregation results.

```sql
SELECT device, max(cpu_usage)
FROM tumble(devices, timestamp, 5s)
GROUP BY device, window_end
EMIT AFTER WATERMARK DELAY 2s;
```

Same as the above delayed tumble window aggregation, except in this query, user specifies a **specific time column** `timestamp` for tumble windowing.

The example below is so called processing time processing which uses wall clock time to assign windows. Timeplus internally processes `now/now64` in a streaming way.

```sql
SELECT device, max(cpu_usage)
FROM tumble(devices, now64(3, 'UTC'), 5s)
GROUP BY device, window_end
EMIT AFTER WATERMARK DELAY 2s;
```

### Hop Streaming Window Aggregation {#hop}

Like [Tumble](#tumble), Hop also slices the unbounded streaming data into smaller windows, and it has an additional sliding step.

```sql
SELECT <column_name1>, <column_name2>, <aggr_function>
FROM hop(<table_name>, [<timestamp_column>], <hop_slide_size>, [hop_windows_size], [<time_zone>])
[WHERE clause]
GROUP BY [<window_start | window_end>], ...
EMIT <window_emit_policy>
SETTINGS <key1>=<value1>, <key2>=<value2>, ...
```

Hop window is a more generalized window compared to tumble window. Hop window has an additional
parameter called `<hop_slide_size>` which means window progresses this slide size every time. There are 3 cases:

1. `<hop_slide_size>` is less than `<hop_window_size>`. Hop windows have overlaps meaning an event can fall into several hop windows.
2. `<hop_slide_size>` is equal to `<hop_window_size>`. Degenerated to a tumble window.
3. `<hop_slide_size>` is greater than `<hop_window_size>`. Windows has a gap in between. Usually not useful, hence not supported so far.

Please note, at this point, you need to use the same time unit in `<hop_slide_size>` and `<hop_window_size>`, for example `hop(device_utils, 1s, 60s)` instead of `hop(device_utils, 1s, 1m)`. 

Here is one hop window example which has 2 seconds slide and 5 seconds hop window.

```
["2020-01-01 00:00:00", "2020-01-01 00:00:05]
["2020-01-01 00:00:02", "2020-01-01 00:00:07]
["2020-01-01 00:00:04", "2020-01-01 00:00:09]
["2020-01-01 00:00:06", "2020-01-01 00:00:11]
...
```

Except that the hop window can have overlaps, other semantics are identical to the tumble window.

```sql
SELECT device, max(cpu_usage)
FROM hop(device_utils, 2s, 5s)
GROUP BY device, window_end
EMIT AFTER WATERMARK;
```

The above example SQL continuously aggregates max cpu usage per device per hop window for table `device_utils`. Every time a window is closed, Timeplus emits the aggregation results.

### Last X Streaming Processing

In streaming processing, there is one typical query which is processing the last X seconds / minutes / hours of data. For example, show me the cpu usage per device in the last 1 hour. We call this type of processing `Last X Streaming Processing` in Timeplus and Timeplus provides a specialized SQL extension for ease of use: `EMIT LAST <n><UNIT>`. As in other parts of streaming queries, users can use interval shortcuts here.

**Note** For now, last X streaming processing is process time processing by default and Timeplus will seek streaming storage to backfill data in last X time range and it is using wall clock time to do the seek. Event time based on last X processing is still under development. When event based last X processing is ready, the by default last X processing will be changed to event time.

#### Last X Tail

Tailing events whose event timestamps are in the last X range.

```sql
SELECT <column_name1>, <column_name2>, ...
FROM <table_name>
WHERE <clause>
EMIT LAST INTERVAL <n> <UNIT>;
```

Examples

```sql
SELECT *
FROM device_utils
WHERE cpu_usage > 80
EMIT LAST 5m
```

The above example filters events in the `device_utils` table where `cpu_usage` is greater than 80% and events are appended in the last 5 minutes. Internally, Timeplus seeks streaming storage back to 5 minutes (wall-clock time from now) and tailing the data from there.

#### Last X Global Aggregation

```sql
SELECT <column_name1>, <column_name2>, <aggr_function>
FROM <table_name>
[WHERE clause]
GROUP BY ...
EMIT LAST INTERVAL <n> <UNIT>
SETTINGS max_keep_windows=<window_count>
```

**Note** Internally Timeplus chops streaming data into small windows and does the aggregation in each small window and as time goes, it slides out old small windows to keep the overall time window fixed and keep the incremental aggregation efficient. By default, the maximum keeping windows is 100. If the last X interval is very big and the periodic emit interval is small,
then users will need to explicitly set up a bigger max window : `last_x_interval / periodic_emit_interval`.

Examples

```sql
SELECT device, count(*)
FROM device_utils
WHERE cpu_usage > 80
GROUP BY device
EMIT LAST 1h AND PERIODIC 5s
SETTINGS max_keep_windows=720;
```

#### Last X Windowed Aggregation

```sql
SELECT <column_name1>, <column_name2>, <aggr_function>
FROM <streaming_window_function>(<table_name>, [<time_column>], [<window_size>], ...)
[WHERE clause]
GROUP BY ...
EMIT LAST INTERVAL <n> <UNIT>
SETTINGS max_keep_windows=<window_count>
```

Examples

```sql
SELECT device, window_end, count(*)
FROM tumble(device_utils, 5s)
WHERE cpu_usage > 80
GROUP BY device, window_end
EMIT LAST 1h
SETTINGS max_keep_windows=720;
```

Similarly, we can apply the last X on hopping window.

### Subquery

#### Vanilla Subquery

A vanilla subquery doesn't have any aggregation (this is a recursive definition), but can have arbitrary number of filter predicates, transformation functions.
Some systems call this `flat map`.

Examples

```sql
SELECT device, max(cpu_usage)
FROM (
    SELECT * FROM device_utils WHERE cpu_usage > 80 -- vanilla subquery
) GROUP BY device;
```

Vanilla subquery can be arbitrarily nested until Timeplus's system limit is hit. The outer parent query can be any normal vanilla query or windows aggregation or global aggregation.

Users can also write the query by using Common Table Expression (CTE) style.

```sql
WITH filtered AS(
    SELECT * FROM device_utils WHERE cpu_usage > 80 -- vanilla subquery
)
SELECT device, max(cpu_usage) FROM filtered GROUP BY device;
```

Multiple CTE can be defined in one query, such as

```sql
WITH cte1 AS (SELECT ..),
     cte2 AS (SELECT ..)
SELECT .. FROM cte1 UNION SELECT .. FROM cte2    
```

CTE with column alias is not supported.

#### Streaming Window Aggregated Subquery

A window aggregate subquery contains windowed aggregation. There are some limitations users can do with this type of subquery.

1. Timeplus supports window aggregation parent query over windowed aggregation subquery (hop over hop, tumble over tumble etc), but it only supports 2 levels. When laying window aggregation over window aggregation, please pay attention to the window size: the window
2. Timeplus supports multiple outer global aggregations over a windowed subquery. (Not working for now).
3. Timeplus allows arbitrary flat transformation (vanilla query) over a windows subquery until a system limit is hit.

Examples

```sql
-- tumble over tumble
WITH avg_5_second AS (
    SELECT device, avg(cpu_usage) AS avg_usage, any(window_start) AS start -- tumble subquery
    FROM
      tumble(device_utils, 5s)
    GROUP BY device, window_start
)
SELECT device, max(avg_usage), window_end -- outer tumble aggregation query
FROM tumble(avg_5_second, start, 10s)
GROUP BY device, window_end;
```

```sql
-- global over tumble
SELECT device, max(avg_usage) -- outer global aggregation query
FROM
(
    SELECT device, avg(cpu_usage) AS avg_usage -- tumble subquery
    FROM
        tumble(device_utils, 5s)
    GROUP BY device, window_start
) AS avg_5_second
GROUP BY device;
```

#### Global Aggregated Subquery

A global aggregated subquery contains global aggregation. There are some limitations users can do with global aggregated subquery:

1. Timeplus supports global over global aggregation and there can be multiple levels until a system limit is hit.
2. Flat transformation over global aggregation can be multiple levels until a system limit is hit.
3. Window aggregation over global aggregation is not supported.

Examples

```sql
SELECT device, max_k(avg_usage,5) -- outer global aggregation query
FROM
(
    SELECT device, avg(cpu_usage) AS avg_usage -- global aggregation subquery
    FROM device_utils
    GROUP BY device
) AS avg_5_second;
```

### JOINs

Please check [Joins](joins).
