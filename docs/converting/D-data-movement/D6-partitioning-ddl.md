# D6 · Partitioning and clustering DDL → config

> **Part D — Data movement, DDL, metadata** · Sourcing: `SRC`
> **The question:** my `CREATE TABLE` specifies partitioning. Where does it go?

Into `config`. Direct mapping, and it's one of the few conversions with no
subtlety — except for what happens when you change it later.

## The mapping

```sql
CREATE TABLE analytics.daily_events (
    event_date DATE,
    user_id INT64,
    event_count INT64
)
PARTITION BY event_date
CLUSTER BY user_id
OPTIONS (
    partition_expiration_days = 90,
    require_partition_filter = TRUE
);
```

```sql
{{ config(
    materialized='incremental',
    partition_by={
        'field': 'event_date',
        'data_type': 'date',
        'granularity': 'day'
    },
    cluster_by=['user_id'],
    partition_expiration_days=90,
    require_partition_filter=true
) }}
```

## The `partition_by` dict

```python
field: str
data_type: str = "date"
granularity: str = "day"
range: Optional[Dict[str, Any]] = None
time_ingestion_partitioning: bool = False
copy_partitions: bool = False
```

String values are **lowercased at parse**, so `'DATE'` and `'date'` are the same.
Full detail in
[the balanced track](../../balanced/06-partition-config.md).

Common forms:

```sql
-- daily date
partition_by={'field': 'event_date', 'data_type': 'date', 'granularity': 'day'}

-- hourly timestamp
partition_by={'field': 'created_at', 'data_type': 'timestamp', 'granularity': 'hour'}

-- integer range
partition_by={'field': 'customer_id', 'data_type': 'int64',
              'range': {'start': 0, 'end': 1000000, 'interval': 1000}}
```

## Read the live DDL, not the script

The script's `CREATE` may not have run in years. If someone altered the table
since, the script no longer describes it:

```sql
select ddl from `project.analytics.INFORMATION_SCHEMA.TABLES`
where table_name = 'daily_events';
```

Build your config from that. This is [A5](../A-assess/A5-hidden-state.md) applied
to physical layout, and getting it wrong is how a conversion silently drops
clustering.

## Changing it later is destructive

The one thing to know beyond the mapping.

BigQuery won't replace a table with one that has a different partitioning spec.
dbt checks with `adapter.is_replaceable`, and on a mismatch logs
`Hard refreshing <relation> because it is not replaceable` and issues a **`DROP`**
before the `CREATE`.

So changing `partition_by` or `cluster_by` on an existing model turns the next
`--full-refresh` into drop-and-recreate. Grants are reapplied afterwards; anything
else attached to the old table isn't, and there's a window where the table doesn't
exist. See
[the balanced track](../../balanced/01-how-the-materialization-runs.md#full-refresh-can-silently-become-a-drop).

Note also that `_partitions_match` compares the **field and granularity**, not
`data_type`, for time partitioning.

## `require_partition_filter` doesn't do what you hope

Setting it protects you from *other people's* unbounded queries. It does not make
your own incremental model cheap — dbt satisfies the requirement with a tautology
that restricts nothing:

```sql
(DBT_INTERNAL_DEST.event_date is null or DBT_INTERNAL_DEST.event_date is not null)
```

You still need a real bound in `incremental_predicates` —
[B12](../B-write-patterns/B12-extra-predicates.md).

## Granularity: check the arithmetic

BigQuery allows 4,000 partitions per table. Hourly hits that in about five and a
half months; daily takes eleven years.

If the script sharded or partitioned hourly, ask whether it needed to. Converting
hourly to daily during consolidation is often right, and it's much easier to do
now than after a year of data — [D5](D5-sharded-tables.md).

---

Previous: [D5 · Date-sharded tables](D5-sharded-tables.md) ·
Next: [D7 · Expiration, labels, description](D7-table-options.md)
