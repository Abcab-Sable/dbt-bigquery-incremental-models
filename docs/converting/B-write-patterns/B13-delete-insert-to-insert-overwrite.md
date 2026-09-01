# B13 · `DELETE` + `INSERT` → `insert_overwrite`

> **Part B — Write-pattern archetypes** · Sourcing: `SRC`
> **The question:** my script clears a date range and reloads it. What's the dbt equivalent?

The most common script shape in analytics, and the conversion with the most
dangerous edge. This page does the mapping. **Read
[B14](../BACKLOG.md#part-b--write-pattern-archetypes) immediately afterwards** —
the naive version of this conversion is silently wrong in a way your script was
not.

## The starting point

```sql
DECLARE start_date DATE DEFAULT DATE_SUB(CURRENT_DATE(), INTERVAL 3 DAY);

DELETE FROM analytics.daily_events
WHERE event_date >= start_date;

INSERT INTO analytics.daily_events (event_date, user_id, event_count)
SELECT
    event_date,
    user_id,
    COUNT(*) AS event_count
FROM raw.events
WHERE event_date >= start_date
GROUP BY event_date, user_id;
```

Clear the last three days, recompute them, put them back. Simple and correct.

## What it becomes

```sql
{{ config(
    materialized='incremental',
    incremental_strategy='insert_overwrite',
    partition_by={
        'field': 'event_date',
        'data_type': 'date',
        'granularity': 'day'
    }
) }}

select
    event_date,
    user_id,
    count(*) as event_count
from {{ source('raw', 'events') }}
{% if is_incremental() %}
  where event_date >= date_sub(current_date(), interval 3 day)
{% endif %}
group by event_date, user_id
```

The `DELETE` disappears entirely. That's the whole idea of `insert_overwrite`:
you declare what a partition should contain, and dbt clears it before writing.

## The mapping

| In your script | In dbt |
| --- | --- |
| `DELETE FROM ... WHERE event_date >= X` | **gone** — implied by the strategy |
| the `INSERT`'s `SELECT` | the model body |
| `WHERE event_date >= X` in the `SELECT` | the `is_incremental()` block |
| `DECLARE start_date ...` | inline, or a var ([E6](../BACKLOG.md#part-e--statement-level-translation)) |
| the column the `DELETE` filters on | **`partition_by.field`** |

That last row is the one to get right. **The column your `DELETE` filtered on is
almost always your partition column.** Your script already told you how the data
is sliced; the config just makes it explicit.

## `partition_by` is mandatory here

Without it, compilation fails before any SQL runs:

> The 'insert_overwrite' strategy requires the `partition_by` config.

If the target table isn't already partitioned on that column, you have a
migration on your hands, not just a conversion — the table must be rebuilt.
Expect `--full-refresh` to **drop and recreate** rather than replace, because
`is_replaceable` returns false when the partitioning spec changes. See
[the balanced track](../../balanced/01-how-the-materialization-runs.md#full-refresh-can-silently-become-a-drop).

Check what you've got before you start:

```sql
select table_name, ddl
from `your_project.your_dataset.INFORMATION_SCHEMA.TABLES`
where table_name = 'daily_events';
```

## Granularity must match your delete

If your `DELETE` worked in whole days, partition by `day`. If it cleared whole
months, partition by `month`. A mismatch means the unit dbt replaces isn't the
unit your script cleared, and the conversion changes behaviour at the edges.

Watch for hourly. Scripts that delete by hour are common, and hourly
partitioning hits BigQuery's 4,000-partition limit in about five and a half
months. If the script deletes hourly but the table only needs daily granularity,
partition by `day` and let each run rebuild whole days.

## What dbt actually generates

Worth seeing, because it explains the edge case:

```sql
declare dbt_partitions_for_replacement array<date>;

create or replace table <tmp> as ( <your select> );

set (dbt_partitions_for_replacement) = (
    select as struct array_agg(distinct event_date IGNORE NULLS) from <tmp>
);

merge into analytics.daily_events as DBT_INTERNAL_DEST
    using (select * from <tmp>) as DBT_INTERNAL_SOURCE
    on FALSE
when not matched by source
     and date(DBT_INTERNAL_DEST.event_date) in unnest(dbt_partitions_for_replacement)
    then delete
when not matched then insert (...) values (...);

drop table if exists <tmp>;
```

Compare that to your script. Both delete a range and insert. But look at where
the range comes from.

**Your script's range came from `start_date`** — a value you declared,
independent of the data.

**dbt's range comes from `array_agg(distinct event_date)` over the rows your
model produced.** It's derived from output, not declared.

When your model produces rows for all three days, those are equivalent. When it
doesn't, they are not — and that is [B14](../BACKLOG.md#part-b--write-pattern-archetypes).

## Static partitions: closer to what your script did

You can restore the declared behaviour by naming the partitions:

```sql
{{ config(
    materialized='incremental',
    incremental_strategy='insert_overwrite',
    partition_by={'field': 'event_date', 'data_type': 'date', 'granularity': 'day'},
    partitions=[
        'date_sub(current_date(), interval 2 day)',
        'date_sub(current_date(), interval 1 day)',
        'current_date()'
    ]
) }}
```

Now the predicate is built from your literals rather than from the data, which is
exactly what `DELETE WHERE event_date >= start_date` did.

Two cautions. The values are interpolated **verbatim** — they're SQL expressions,
so `current_date()` works but a fixed date needs its own quotes (`'2026-08-31'`).
And never build this list from untrusted input; there is no escaping.

**If your script's range could ever be empty, use this form.** That decision is
[B14](../BACKLOG.md#part-b--write-pattern-archetypes), and it's not optional.

## Before you retire the script

- [ ] Target table is partitioned on the delete column, at matching granularity
- [ ] `dbt compile` output shows the partition predicate you expect
- [ ] You've decided static vs dynamic, deliberately, having read B14
- [ ] Row counts match for a range both versions have processed ([H2](../BACKLOG.md#part-h--proving-correctness))
- [ ] You've tested a range that produces **zero rows**, and confirmed the result
      is what you want

That last box is the one that catches the conversion bug. Don't tick it from
memory.

---

Previous: [B8 · The `MERGE` `ON` clause → `unique_key`](B8-merge-on-clause-to-unique-key.md) ·
Next in wave 1: **B14 · When the range can legitimately empty** *(not yet written)* ·
Back to [the backlog](../BACKLOG.md)
