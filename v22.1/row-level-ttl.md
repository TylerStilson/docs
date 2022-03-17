---
title: Batch Delete Expired Data with Row-Level TTL
summary: Automatically delete rows of expired data older than a specified interval.
toc: true
keywords: ttl,delete,deletion,bulk deletion,bulk delete,expired data,expire data,time to live,row-level ttl,row level ttl
docs_area: develop
---

{% include {{page.version.version}}/sql/row-level-ttl.md %}

Writing and managing scheduled jobs from the application layer to mark rows as expired and perform the necessary deletions can become complicated due to the need to avoid negative performance impact on foreground application queries.

By using Row-Level TTL, you can avoid the complexity of balancing the timeliness of the deletions vs. the effect of those deletions on database performance when processing foreground traffic from your application.

Use cases for row-level TTL include:

- Delete inactive data events to manage data size and performance: For example, you may want to delete order records from an online store after 90 days.

- Delete data no longer needed for compliance: For example, a banking application may need to keep some subset of data for a period of time due to financial regulations. Row-Level TTL can be used to remove data older than that period on a rolling, continuous basis.

- Outbox pattern: When events are written to an outbox table and published to an external system like [Kafka](https://en.wikipedia.org/wiki/Apache_Kafka), those events must be deleted to prevent unbounded growth in the size of the outbox table.

## How it works

At a high level, Row-Level TTL works by:

1. Issuing a [selection query](selection-queries.html) at a [historical timestamp](as-of-system-time.html), yielding a set of rows that are eligible for deletion.
1. Given the set of rows eligible for deletion, split a logical [`DELETE`](delete.html) statement covering those rows into multiple concrete SQL queries that will have the equivalent effect. Each of these SQL queries will operate on a subset of the eligible rows.
1. As part of the above process, deciding how many rows to [`SELECT`](select-clause.html) and [`DELETE`](delete.html) at once in each of the above queries.
1. Running the SQL queries described above in parallel as [background jobs](show-jobs.html).
1. All the while applying a rate limit to control the rate of those background deletion queries to minimize the performance impact on foreground application queries.

The process above is conceptually similar to the process described by [Batch delete on an indexed column](bulk-delete-data.html#batch-delete-on-an-indexed-column), except that Row-Level TTL is built into CockroachDB, so it saves you from having to write code to manage the process from your application and/or external job processing framework, including tuning the rate and performance of your background queries so they don't affect foreground application query performance.

### Syntax overview

TTLs are defined on a per-table basis using SQL statements. The syntax for creating a table with an automatically managed TTL extends the [`storage_parameter` syntax](sql-grammar.html#opt_with_storage_parameter_list). For example, the SQL statement

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

To see the hidden column and the storage parameters on the `events` table, enter the [`SHOW CREATE TABLE`](show-create.html) statement:

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

### TTL storage parameters

The settings that control CockroachDB's TTL are provided using [storage parameters](sql-grammar.html#opt_with_storage_parameter_list). These parameters can be set during table creation using [`CREATE TABLE`](#create-a-table-with-row-level-ttl), or added to an existing table using the [`ALTER TABLE`](#add-row-level-ttl-to-an-existing-table) statement.

| Description                                      | Option                                                                                                                                                                       |
|--------------------------------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `ttl_expire_after`                               | The [interval](interval.html) when a TTL will expire. Default: '30 days'. Minimum value: '5 minutes'.                                                                        |
| `ttl` <a name="param-ttl"></a>                   | Signifies if a TTL is active. Automatically set.                                                                                                                             |
| `ttl_select_batch_size`                          | How many rows to [select](select-clause.html) at one time from the set of expired rows. Default: 500. Minimum: 1.                                                            |
| `ttl_delete_batch_size`                          | How many rows to [delete](delete.html) at a time. Default: 100. Minimum: 1.                                                                                                  |
| `ttl_delete_rate_limit`                          | Maximum number of rows to be deleted per second (rate limit). Default: 0 (no limit).                                                                                         |
| `ttl_range_concurrency`                          | How many concurrent batches of expired rows are being worked on at a time. Default: `cpu_core_count`. Minimum: 1.                                                            |
| `ttl_row_stats_poll_interval`                    | If set, counts rows and expired rows on the table to report as Prometheus metrics while the TTL job is running. Unset by default, meaning no stats are fetched and reported. |
| `ttl_pause`                                      | If set, stops the TTL job from executing.                                                                                                                                    |
| `ttl_job_cron` <a name="param-ttl-job-cron"></a> | Frequency at which the TTL job runs, specified using [CRON syntax](https://cron.help).                                                                                       |
| `ttl_automatic_column`                           | If set, use the value of the `crdb_internal_expiration` hidden column. This is unset when `ttl_expiration_expression` is used.                                               |
|                                                  |                                                                                                                                                                              |

### The deletion job

Once rows are expired (that is, have crossed the TTL), they are _eligible_ to be deleted. However, they may not be deleted right away - instead, they are scheduled for deletion using a job that is run at the interval defined by the `ttl_job_cron` [storage parameter](#ttl-storage-parameters).

## Examples

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

XXX: DOES THIS SYNTAX WORK?

{% include_cached copy-clipboard.html %}
~~~ sql
ALTER TABLE events SET (ttl_expire_after = '90 days');
~~~

### Remove row-level TTL from a table

To drop the TTL on an existing table, reset the [`ttl` storage parameter](#param-ttl).

{% include_cached copy-clipboard.html %}
~~~ sql
ALTER TABLE events RESET (ttl)
~~~

### Reset a TTL storage parameter to its default value

To reset a [TTL storage parameter](#ttl-storage-parameters) to its default value, use the [`ALTER TABLE`](alter-table.html) statement:

{% include_cached copy-clipboard.html %}
~~~ sql
ALTER TABLE events RESET (ttl_job_cron);
~~~

If you try to reset a TTL storage parameter in an invalid order, CockroachDB will signal an error. For example, there is only one way to [remove row-level TTL from a table](#remove-row-level-ttl-from-a-table). If you try to remove the TTL from a table by resetting the `ttl_expire_after` storage parameter you set earlier, an error is signalled:

{% include_cached copy-clipboard.html %}
~~~ sql
ALTER TABLE tbl RESET (ttl_expire_after);
~~~

~~~
resetting "ttl_expire_after" is not permitted
HINT: use `RESET (ttl)` to remove TTL from the table
~~~

### View TTL jobs

XXX: SHOULD WE FILTER THIS FURTHER?

{% include_cached copy-clipboard.html %}
~~~ sql
SELECT * FROM [SHOW SCHEDULES] WHERE label LIKE 'row-level-ttl-%';
~~~

### View TTL storage parameters on a table

{% include_cached copy-clipboard.html %}
~~~ sql
SELECT relname, reloptions FROM pg_class WHERE relname = 'events';
~~~

### Control how often the TTL job runs

Setting a TTL on a table controls when the rows therein are considered expired, but it only says that such rows _may_ be deleted at any time after the expiration. To control how often the TTL deletion job runs, use the [`ttl_job_cron` storage parameter](#param-ttl-job-cron), which supports [CRON syntax](https://cron.help).

To control the job interval at [`CREATE TABLE`](create-table.html) time, add the storage parameter as shown below:

{% include_cached copy-clipboard.html %}
~~~ sql
CREATE TABLE tbl (
  id UUID PRIMARY KEY default gen_random_uuid(),
  value TEXT
) WITH (ttl_job_cron = '@daily')
~~~

To update the TTL deletion job interval on a table that already has Row-Level TTL enabled, use [`ALTER TABLE`](alter-table.html):

{% include_cached copy-clipboard.html %}
~~~ sql
ALTER TABLE tbl SET (ttl_job_cron = '@weekly')
~~~

### Reset a storage parameter to its default value

To reset a storage parameter to its default value, use [`ALTER TABLE`](alter-table.html):

{% include_cached copy-clipboard.html %}
~~~ sql
ALTER TABLE tbl RESET (ttl_job_cron)
~~~

### Filter out expired rows from a selection query

XXX: WRITE ME

... use the `crdb_internal_expiration` hidden column?

## Common errors

If you attempt to update a [TTL-related storage parameter](#ttl-storage-parameters) on a table that does not have TTL enabled, you will see the error below:

~~~ 
"ttl_expire_after" must be set
~~~

it is like

## Limitations

XXX: ADD THESE TO KNOWN LIMITATIONS

- You cannot use [foreign keys](foreign-key.html) to create references to or from a table that uses row-level TTL. 
- [`SELECT`](selection-queries.html) queries against tables with Row-Level TTL enabled do not filter out expired rows from the result set. This may be added in a future release.

## See also

- [Bulk-delete Data](bulk-delete-data.html)
- [SQL Performance Best Practices](performance-best-practices-overview.html)
- [Developer Guide Overview](developer-guide-overview.html)
