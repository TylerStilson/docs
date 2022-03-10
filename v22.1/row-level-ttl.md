---
title: Batch Delete Expired Data with Row-Level TTL
summary: Automatically delete rows of expired data older than a specified interval.
toc: true
keywords: ttl,delete,deletion,bulk deletion,bulk delete,expired data,expire data,time to live,row-level ttl,row level ttl
docs_area: develop
---

{% include {{page.version.version}}/sql/row-level-ttl.md %}

By using row-level TTL, you can avoid the complexity of writing and managing scheduled jobs from your application to perform the deletes. This can become complicated due to the need to balance the timeliness of the deletions vs. the effect on database performance when processing foreground traffic from your application.

Use cases for row-level TTL include:

- Delete inactive data events to manage data size and performance: For example, you may want to delete order records from an online store after 90 days.

- Delete data no longer needed for compliance: For example, a banking application may need to keep some subset of data for a period of time due to financial regulations. A TTL can be used to remove data older than that period on a continuous basis.

- Outbox pattern: When events are written to an outbox table and published to a system like Kafka, those events must be deleted to prevent unbounded growth in the size of the outbox table. The TTL can be useful for this.

## How it works

At a high level, Row-Level TTL works by doing approximately the same things you would do yourself if you had to implement this feature in your application. For example, if you were implementing batch deletes of expired data yourself, you might do something like the following:

1. Issue a [selection query](selection-queries.html) at a [historical timestamp](as-of-system-time.html), yielding a set of rows that are eligible for deletion.
2. Rather than issue a naive [`DELETE`](delete.html) on all the rows that are eligible for deletion, which would likely have [bad performance implications](delete.html#preserving-delete-performance-over-time), write a batch-delete loop similar to that described by [Batch delete on an indexed column](bulk-delete-data.html#batch-delete-on-an-indexed-column). This batch-delete loop runs in the background and operates on a subset of the eligible rows per iteration to protect the performance of foreground application queries.
3. Depending on the performance impact of the previous step, you'd need to tweak the size of the batches per iteration, and limit the rate of those iterations, to find the best tradeoff you can between the timeliness of the deletions vs. the effect on overall database performance.

In other words, Row-Level TTL handles:

- Sharding the delete into multiple queries, and running those queries in parallel as background jobs
- Deciding how many rows to SELECT and DELETE at once
- Rate-limiting to control the rate of SELECTs and DELETEs

### Table metadata and storage parameters

TTLs are defined on a per-table basis. The syntax for creating a table with an automatically managed TTL extends the [`storage_parameter` syntax](sql-grammar.html#opt_with_storage_parameter_list).

For example, the SQL statement

{% include_cached copy-clipboard.html %}
~~~ sql
CREATE TABLE events (
  id UUID PRIMARY KEY default gen_random_uuid(),
  description TEXT,
  inserted_at TIMESTAMP default current_timestamp()
) WITH (ttl_expire_after = '3 minutes');
~~~

has the following effects:

1. Creates a repeating [scheduled job](#view-ttl-jobs) for the `events` table.
2. Adds a hidden column `crdb_internal_expiration` of type [`TIMESTAMPTZ`](timestamp.html) to represent the TTL.
3. Implicitly adds the `ttl` and `ttl_automatic_column` [storage parameters](#ttl-storage-parameters).

To see the hidden column and the storage parameters, enter the [`SHOW CREATE TABLE`](show-create.html) statement:

{% include_cached copy-clipboard.html %}
~~~ sql
SHOW CREATE TABLE events;
~~~

~~~
  table_name |                                                                                           create_statement
-------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  events     | CREATE TABLE public.events (
             |     id UUID NOT NULL DEFAULT gen_random_uuid(),
             |     description STRING NULL,
             |     inserted_at TIMESTAMP NULL DEFAULT current_timestamp():::TIMESTAMP,
             |     crdb_internal_expiration TIMESTAMPTZ NOT VISIBLE NOT NULL DEFAULT current_timestamp():::TIMESTAMPTZ + '00:03:00':::INTERVAL ON UPDATE current_timestamp():::TIMESTAMPTZ + '00:03:00':::INTERVAL,
             |     CONSTRAINT events_pkey PRIMARY KEY (id ASC)
             | ) WITH (ttl = 'on', ttl_automatic_column = 'on', ttl_expire_after = '00:03:00':::INTERVAL)
(1 row)

~~~

{% include_cached copy-clipboard.html %}
~~~ sql
INSERT INTO events (description) VALUES ('a thing', 'another thing', 'yet another thing');
~~~

~~~
INSERT 3
~~~

### TTL storage parameters

The settings that control CockroachDB's TTL are provided using [storage parameters](sql-grammar.html#opt_with_storage_parameter_list).

XXX: REVISE CONTENTS OF TABLE

| Option                        | Description                                                                                                                                               |
|-------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------|
| `ttl`                         | Automatically set option. Signifies if a TTL is active. Not used for the job.                                                                             |
| `ttl_automatic_column`        | Automatically set option if automatic connection is enabled. Not used for the job.                                                                        |
| `ttl_expire_after`            | When a TTL would expire. Accepts any interval. Defaults to ''30 days''. Minimum of `'5 minutes'`.                                                         |
| `ttl_expiration_expression`   | If set, uses the expression specified as the TTL expiration. Defaults to just using the `crdb_internal_expiration` column.                                |
| `ttl_select_batch_size`       | How many rows to fetch from the range that have expired at a given time. Defaults to 500. Must be at least `1`.                                           |
| `ttl_delete_batch_size`       | How many rows to delete at a time. Defaults to 100. Must be at least `1`.                                                                                 |
| `ttl_range_concurrency`       | How many concurrent ranges are being worked on at a time. Defaults to `cpu_core_count`. Must be at least `1`.                                             |
| `ttl_delete_rate_limit`       | Maximum number of rows to be deleted per second (acts as the rate limit). Defaults to 0 (signifying none).                                                |
| `ttl_row_stats_poll_interval` | Whilst the TTL job is running, counts rows and expired rows on the table to report as prometheus metrics. By default unset, meaning no stats are fetched. |
| `ttl_pause`                   | Stops the TTL job from executing.                                                                                                                         |
| `ttl_job_cron`                | Frequency the job runs, specified using the CRON syntax.                                                                                                  |

### The deletion job

Once rows are expired (that is, have crossed the TTL), they are _eligible_ to be deleted. However, they may not be deleted right away - instead, they are scheduled for deletion using a job that is run at the interval defined by the `ttl_job_cron` [storage parameter](#ttl-storage-parameters).

## Examples

### View TTL jobs

{% include_cached copy-clipboard.html %}
~~~ sql
SELECT * FROM [SHOW SCHEDULES] WHERE label LIKE 'row-level-ttl-%';
~~~

### Create a table with row-level TTL

To specify a TTL when creating a table, use the SQL syntax shown below. For example, to create a new table with rows that expire after 15 minutes, execute a statement like the following:

{% include_cached copy-clipboard.html %}
~~~ sql
CREATE TABLE events (
  id UUID PRIMARY KEY,
  description TEXT
) WITH (ttl_expire_after = '15 minutes');
~~~

### Add row-level TTL to an existing table

To add a TTL to an existing table, use the SQL syntax shown below.

{% include_cached copy-clipboard.html %}
~~~ sql
ALTER TABLE events SET (ttl_expire_after = '15 minutes');
~~~

### Drop row-level TTL from a table

To drop the TTL on an existing table, 

## Limitations

XXX: ADD THESE TO KNOWN LIMITATIONS

- You cannot use [foreign keys](foreign-key.html) to create references to or from a table that uses row-level TTL. 

## See also

- [Bulk-delete Data](bulk-delete-data.html)
- [SQL Performance Best Practices](performance-best-practices-overview.html)
- [Developer Guide Overview](developer-guide-overview.html)
