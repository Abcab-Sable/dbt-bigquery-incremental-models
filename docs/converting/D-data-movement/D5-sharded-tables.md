# D5 · Date-sharded tables → one partitioned table

> **Part D — Data movement, DDL, metadata** · Sourcing: `CRAFT`
> **The question:** we have `events_20260831` and 900 friends. What now?

Consolidate them into one partitioned table. It's the highest-value structural
change available during most conversions, and BigQuery has recommended it over
sharding for years.

## Why sharding exists

Date-sharded tables predate BigQuery's partitioning support. Creating
`events_YYYYMMDD` was the only way to limit how much a query read. Scripts written
in that era shard by habit, and the habit outlived the reason.

## What it costs you now

**A 4,000-table dataset limit approaches**, and metadata operations get slow.
Listing a dataset with 900 tables is unpleasant; with 3,000 it's worse.

**Every query needs a wildcard**, and every wildcard needs a `_TABLE_SUFFIX`
filter that someone will forget — [D4](D4-wildcard-tables.md).

**Schema drift is invisible.** Nothing keeps 900 tables consistent, so they
diverge, and the divergence surfaces as confusing wildcard behaviour.

**dbt models it badly.** One relation per shard, or one vague wildcard source.
Neither is good lineage.

**Per-table overhead.** Metadata, storage minimums, and query planning across
hundreds of tables cost more than one partitioned table.

## The consolidation

One model reads the shards and writes a partitioned table:

```sql
-- models/staging/stg_events.sql
{{ config(
    materialized='incremental',
    incremental_strategy='insert_overwrite',
    partition_by={'field': 'event_date', 'data_type': 'date', 'granularity': 'day'}
) }}

select
    parse_date('%Y%m%d', _table_suffix) as event_date,
    event_id,
    user_id,
    payload
from {{ source('raw', 'events_shards') }}
{% if is_incremental() %}
  where _table_suffix >= format_date('%Y%m%d', date_sub(current_date(), interval 3 day))
{% endif %}
```

Then everything downstream reads `ref('stg_events')` and never sees a wildcard
again.

## Backfilling the history

The first build reads every shard, which may be large. Options:

**One full run**, if it fits in time and budget. Simplest.

**Microbatch**, if it doesn't:

```sql
{{ config(
    materialized='incremental',
    incremental_strategy='microbatch',
    event_time='event_date',
    batch_size='day',
    begin='2024-01-01',
    partition_by={'field': 'event_date', 'data_type': 'date', 'granularity': 'day'}
) }}
```

Each batch handles one day, retryable independently —
[the microbatch page](../../balanced/05-microbatch.md).

**Manual ranges**, running `--vars` over chunks —
[G7](../G-scheduling/G7-backfill-partition-ranges.md).

## Handle the schema drift

You will find it. Shards from different eras have different columns, and a
`select *` over the wildcard takes the first shard's schema.

List columns explicitly and coalesce what's missing:

```sql
select
    parse_date('%Y%m%d', _table_suffix) as event_date,
    event_id,
    user_id,
    -- added 2025-03; null in earlier shards
    cast(null as string) as consent_version
from ...
```

Better: handle the eras separately and `union all`, so each era's real schema is
explicit. Verbose, and it documents the history rather than hiding it.

## Don't delete the shards yet

The consolidated table is a new thing. Keep the shards until:

- parity is proven ([H2](../H-verification/H2-row-count-parity.md),
  [H3](../H-verification/H3-checksum-parity.md))
- every consumer of the wildcard has moved
- the rollback window has passed —
  [I6](../I-migration/I6-rollback-keeping-script.md)

Then delete them, and enjoy the dataset listing.

## If the loader still writes shards

Common: an ingestion process you don't control keeps creating `events_YYYYMMDD`.

Fine. The staging model keeps consolidating incrementally, reading only recent
shards each run. The shards become an implementation detail of ingestion, and
everything in your project reads the partitioned table.

Changing the loader is a separate piece of work — and not one to do during a
conversion ([K11](../K-antipatterns/K11-convert-and-optimise.md)).

---

Previous: [D4 · Wildcard tables and `_TABLE_SUFFIX`](D4-wildcard-tables.md) ·
Next: [D6 · Partitioning and clustering DDL](D6-partitioning-ddl.md)
