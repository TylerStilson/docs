---
title: Hash-sharded Indexes
summary: Hash-sharded indexes can eliminate single-range hotspots and improve write performance on sequentially-keyed indexes at a small cost to read performance
toc: true
docs_area: develop
---

If you are working with a table that must be indexed on sequential keys, you should use **hash-sharded indexes**. Hash-sharded indexes distribute sequential traffic uniformly across ranges, eliminating single-range hotspots and improving write performance on sequentially-keyed indexes at a small cost to read performance.

## How hash-sharded indexes work

CockroachDB automatically splits ranges of data in [the key-value store](architecture/storage-layer.html) based on [the size of the range](architecture/distribution-layer.html#range-splits) and on [the load streaming to the range](load-based-splitting.html). To split a range based on load, the system looks for a point in the range that evenly divides incoming traffic. If the range is indexed on a column of data that is sequential in nature (e.g., [an ordered sequence](sql-faqs.html#what-are-the-differences-between-uuid-sequences-and-unique_rowid) or a series of increasing, non-repeating [`TIMESTAMP`s](timestamp.html)), then all incoming writes to the range will be the last (or first) item in the index and appended to the end of the range. As a result, the system cannot find a point in the range that evenly divides the traffic, and the range cannot benefit from load-based splitting, creating a hotspot on the single range.

Hash-sharded indexes contain a [virtual computed column](computed-columns.html#virtual-computed-columns), known as a shard column. CockroachDB uses this shard column, as opposed to the sequential column in the index, to distribute the rows evenly upon insert across the nodes of your cluster, thereby eliminating hotspots. The shard column is hidden by default but can be seen with [`SHOW COLUMNS`](show-columns.html).

For details about the mechanics and performance improvements of hash-sharded indexes in CockroachDB, see our [Hash Sharded Indexes Unlock Linear Scaling for Sequential Workloads](https://www.cockroachlabs.com/blog/hash-sharded-indexes-unlock-linear-scaling-for-sequential-workloads/) blog post.

{{site.data.alerts.callout_info}}
Hash-sharded indexes created in v22.1 will not [backfill](use-changefeeds.html#schema-changes-with-column-backfill), as the shard column isn't stored. Hash-sharded indexes created prior to v22.1 will backfill if `schema_change_policy` is set to `backfill`, as they use a stored column. If you wish for CockroachDB not to backfill your hash-sharded indexes created prior to v22.1, drop them and recreate them.
{{site.data.alerts.end}}

### Performance implications with hash-sharded indexes

The main performance benefits with hash-sharded indexes are seen at write time, eliminating hotspots by distributing sequential data across multiple nodes within your cluster. The trade-off to this, however, is a small performance impact on reading sequential data, as it's not guaranteed that sequentially close values will be on the same node.

### Hash-sharded indexes on partitioned tables

You can create hash-sharded indexes with implicit partitioning under the following scenarios:

- The table is partitioned implicitly with [`REGIONAL BY ROW`](multiregion-overview.html#regional-by-row-tables), and the `crdb_region` column is not part of the columns in the hash-sharded index.
- The table is partitioned implicitly with `PARTITION ALL BY`, and the partition columns are not part of the columns in the hash-sharded index.

However, if an index of a table, whether it be a primary key or secondary index, is explicitly partitioned with `PARTITION BY`, then that index cannot be hash-sharded. Paritioning columns cannot be placed explicitly as key columns of a hash-sharded index as well, including `REGIONAL BY ROW` table's `crdb_region` column.

## Create a hash-sharded index

The general process of creating a hash-sharded index is to add the `USING HASH` clause to one of the following statements:

- [`CREATE INDEX`](create-index.html)
- [`CREATE TABLE`](create-table.html)
- [`ALTER PRIMARY KEY`](alter-primary-key.html)

When this clause is used, CockroachDB creates a computed shard column and then stores each index shard in the underlying key-value store with one of the computed column's hash as its prefix.

## Examples

### Create a table with a hash-sharded primary key

{% include {{page.version.version}}/performance/create-table-hash-sharded-primary-index.md %}

### Create a table with a hash-sharded secondary index

{% include {{page.version.version}}/performance/create-table-hash-sharded-secondary-index.md %}

### Create a hash-sharded secondary index on an existing table

{% include {{page.version.version}}/performance/create-index-hash-sharded-secondary-index.md %}

### Alter an existing primary key to use hash sharding

{% include {{page.version.version}}/performance/alter-primary-key-hash-sharded.md %}

### Show hash-sharded index in `SHOW CREATE TABLE`

Following the above [example](#create-a-hash-sharded-secondary-index-on-an-existing-table), you can show the hash-sharded index definition along with the table creation statement using `SHOW CREATE TABLE`:

{% include copy-clipboard.html %}
~~~ sql
> SHOW CREATE TABLE events;
~~~

~~~
  table_name |                                                        create_statement
-------------+---------------------------------------------------------------------------------------------------------------------------------
  events     | CREATE TABLE public.events (
             |     product_id INT8 NOT NULL,
             |     owner UUID NOT NULL,
             |     serial_number VARCHAR NOT NULL,
             |     event_id UUID NOT NULL,
             |     ts TIMESTAMP NOT NULL,
             |     data JSONB NULL,
             |     crdb_internal_ts_shard_16 INT8 NOT VISIBLE NOT NULL AS (mod(fnv32(crdb_internal.datums_to_bytes(ts)), 16:::INT8)) VIRTUAL,
             |     CONSTRAINT events_pkey PRIMARY KEY (product_id ASC, owner ASC, serial_number ASC, ts ASC, event_id ASC),
             |     INDEX events_ts_idx (ts ASC) USING HASH WITH (bucket_count=16)
             | )
(1 row)
~~~

## See also

- [Indexes](indexes.html)
- [`CREATE INDEX`](create-index.html)
