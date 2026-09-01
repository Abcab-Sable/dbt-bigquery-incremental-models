# J8 · Growing partition counts and the 4,000 limit

> **Part J — Operating it afterwards** · Sourcing: `CRAFT`
> **The question:** we partitioned by hour. When does that become a problem?

In about five and a half months. BigQuery allows **4,000 partitions per table**,
and it's a hard limit — writes start failing when you reach it.

## The arithmetic

| Granularity | Partitions/year | Hits 4,000 after |
| --- | --- | --- |
| `hour` | 8,760 | **5.5 months** |
| `day` | 365 | 10.9 years |
| `month` | 12 | 333 years |
| `year` | 1 | a long time |

Daily is the right answer nearly always. Hourly needs a retention policy from day
one, not from the day it breaks.

## Watch it

```sql
select
    table_name,
    count(distinct partition_id) as partitions,
    min(partition_id) as oldest,
    max(partition_id) as newest
from `project.analytics.INFORMATION_SCHEMA.PARTITIONS`
where table_name = 'daily_events'
group by table_name;
```

Make it a warn-severity test at a threshold you can act on:

```sql
-- tests/daily_events_partition_count.sql
select 1
from (
    select count(distinct partition_id) as n
    from `{{ target.project }}.{{ this.schema }}.INFORMATION_SCHEMA.PARTITIONS`
    where table_name = '{{ this.identifier }}'
)
where n > 3500
```

3,500 gives you months of warning at daily granularity and weeks at hourly.

## The fix is expiry

```sql
{{ config(
    partition_by={'field': 'event_date', 'data_type': 'date', 'granularity': 'day'},
    partition_expiration_days=1095
) }}
```

Old partitions drop automatically. Set it during the conversion — the script
probably had `partition_expiration_days` in its `CREATE` and it's easy to lose
([D7](../D-data-movement/D7-table-options.md)).

Check what the live table has before assuming:

```sql
select option_name, option_value
from `project.analytics.INFORMATION_SCHEMA.TABLE_OPTIONS`
where table_name = 'daily_events';
```

## Changing granularity later is destructive

If you're already hourly and need daily, that's a `partition_by` change — which
means `is_replaceable` returns false and dbt **drops and recreates**
([D6](../D-data-movement/D6-partitioning-ddl.md)).

Which also loses [policy tags and row access
policies](../D-data-movement/D11-policy-tags-rls.md), and gives you a window where
the table doesn't exist.

So the cost of getting granularity wrong is not "change a config" — it's a
migration. Get it right during the conversion, when the table is being rebuilt
anyway.

## Small partitions are slow as well as numerous

The limit isn't the only cost. Thousands of small partitions query *slower* than a
sensible number of larger ones — more metadata, more planning, less efficient
scans.

If your hourly partitions hold a few thousand rows each, daily partitioning with
clustering on the time column will usually be both cheaper and faster.

## While you're checking, check the shape

The partition listing is also a drift signal. A partition that hasn't changed
while its neighbours have, or a gap in the sequence, is worth a look:

```sql
select partition_id, total_rows, last_modified_time
from `project.analytics.INFORMATION_SCHEMA.PARTITIONS`
where table_name = 'daily_events'
order by partition_id desc
limit 30;
```

A recent partition with an old `last_modified_time` is the
[empty-partition](../B-write-patterns/B14-when-the-range-can-empty.md) signature —
the model produced nothing for it and dbt left it alone.

---

Previous: [J7 · Schema evolution over time](J7-schema-evolution.md) ·
Next: [J9 · When to revisit the strategy choice](J9-revisiting-strategy.md)
