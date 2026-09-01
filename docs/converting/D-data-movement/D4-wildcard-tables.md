# D4 · Wildcard tables and `_TABLE_SUFFIX`

> **Part D — Data movement, DDL, metadata** · Sourcing: `CRAFT`
> **The question:** my script queries `events_*`. How does that become a `source()`?

Awkwardly, because a wildcard isn't one table and `source()` names one relation.
You have three options, and the best one is usually to stop using wildcards —
[D5](D5-sharded-tables.md).

## The pattern

```sql
SELECT * FROM `project.raw.events_*`
WHERE _TABLE_SUFFIX BETWEEN '20260801' AND '20260831';
```

`_TABLE_SUFFIX` is the pseudo-column holding whatever the `*` matched. Filtering on
it prunes which tables are read — the sharded-table equivalent of partition
pruning, and just as important for cost.

## Option 1: source with the wildcard in the identifier

```yaml
sources:
  - name: raw
    schema: raw
    tables:
      - name: events_shards
        identifier: "events_*"
```

```sql
select * from {{ source('raw', 'events_shards') }}
where _table_suffix between '20260801' and '20260831'
```

Works, and keeps the reference in the DAG. Caveats: freshness checks don't
meaningfully apply, and the source represents an open-ended set rather than a
relation — so the lineage is honest about the dependency but vague about what it
covers.

**Keep the `_TABLE_SUFFIX` filter.** Without it you read every shard that has ever
existed. This is the single most expensive mistake in wildcard queries, and it
doesn't error.

## Option 2: consolidate first, then read normally

Convert the shards to one partitioned table once, then everything downstream is
ordinary:

```sql
-- models/staging/stg_events.sql
{{ config(
    materialized='incremental',
    incremental_strategy='insert_overwrite',
    partition_by={'field': 'event_date', 'data_type': 'date', 'granularity': 'day'}
) }}

select
    parse_date('%Y%m%d', _table_suffix) as event_date,
    *
from {{ source('raw', 'events_shards') }}
{% if is_incremental() %}
  where _table_suffix >= format_date('%Y%m%d', date_sub(current_date(), interval 3 day))
{% endif %}
```

One model touches the wildcard; everything else reads a proper partitioned table.
This is the recommended shape — see [D5](D5-sharded-tables.md) for why.

## Option 3: enumerate in Jinja

If the shard set is small and known:

```sql
{% set shards = ['20260829', '20260830', '20260831'] %}

{% for s in shards %}
select '{{ s }}' as shard, * from {{ source('raw', 'events_' ~ s) }}
{% if not loop.last %}union all{% endif %}
{% endfor %}
```

Each shard becomes a real DAG edge. Only viable for a fixed, small list — and if
you're generating the list by querying `INFORMATION_SCHEMA`, you're back to
`run_query` and its costs ([C10](../C-structural/C10-dynamic-sql.md)).

## `_TABLE_SUFFIX` is a string

It's always `STRING`, whatever the shards look like. So:

```sql
-- works
where _table_suffix between '20260801' and '20260831'

-- does not prune, and may error
where parse_date('%Y%m%d', _table_suffix) >= '2026-08-01'
```

Wrapping it in a function defeats the pruning, exactly like wrapping a partition
column. Compare strings, and derive the date *after* filtering.

## Schema differences between shards

Wildcard queries take the schema of the **first** table matched. If shards drifted
— a column added in March — earlier shards may not have it, and you get nulls or
errors depending on the direction.

Scripts often carry a `SELECT` listing columns explicitly for exactly this reason.
Preserve that list rather than replacing it with `SELECT *`; the explicit list was
load-bearing.

Check for drift before converting:

```sql
select table_name, count(*) as n_cols
from `project.raw.INFORMATION_SCHEMA.COLUMNS`
where table_name like 'events_%'
group by table_name
order by table_name;
```

Varying `n_cols` means drift, and it's a finding for
[A9](../A-assess/A9-correctness-baseline.md).

---

Previous: [D3 · External tables and BigLake](D3-external-tables.md) ·
Next: [D5 · Date-sharded tables](D5-sharded-tables.md)
