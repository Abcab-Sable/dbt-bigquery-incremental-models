# G8 · Late-arriving data after conversion

> **Part G — Scheduling, parameters, backfills** · Sourcing: `CRAFT`
> **The question:** rows arrive after we've processed their day. Does the conversion handle that?

Only if you make it. And the conversion is the moment to find out, because the
script's handling was usually accidental.

## The two shapes

**Late by event time.** A row belonging to Monday arrives on Wednesday. Its
partition has already been built.

**Late by update.** A row already loaded changes upstream. Its key already exists
downstream.

They need different answers.

## Late by event time

Under `insert_overwrite`, the partition must be **rebuilt** to pick the row up. A
lookback window does that:

```sql
{{ config(
    materialized='incremental',
    incremental_strategy='insert_overwrite',
    partition_by={'field': 'event_date', 'data_type': 'date', 'granularity': 'day'},
    partitions=[
        'date_sub(current_date(), interval 3 day)',
        'date_sub(current_date(), interval 2 day)',
        'date_sub(current_date(), interval 1 day)',
        'current_date()'
    ]
) }}

select ... from {{ source('raw', 'events') }}
{% if is_incremental() %}
  where event_date >= date_sub(current_date(), interval 3 day)
{% endif %}
group by ...
```

Four days rebuilt every run. Anything arriving within three days is picked up;
anything later is not, ever, until someone backfills.

**The window is a decision about your data.** Measure it rather than guessing:

```sql
select
    date_diff(date(_loaded_at), event_date, day) as lag_days,
    count(*) as n
from `raw.events`
where _loaded_at > timestamp_sub(current_timestamp(), interval 30 day)
group by lag_days
order by lag_days;
```

Size the window to cover the tail you care about, and write down why —
otherwise it becomes folklore and someone shrinks it for cost.

## Late by update

Under `merge` with a `unique_key`, this is handled — the row matches and updates,
however late.

Unless `incremental_predicates` bounds the target:

```sql
incremental_predicates=["DBT_INTERNAL_DEST.order_date >= date_sub(current_date(), interval 7 day)"]
```

Now a 30-day-old row can't match anything inside the window and is **inserted as a
duplicate**. Same reasoning as the lookback window: the bound is a claim that
older rows never change — [B12](../B-write-patterns/B12-extra-predicates.md).

## The watermark boundary

A strict `>` filter loses rows sharing the boundary timestamp:

```sql
where updated_at > (select max(updated_at) from {{ this }})
```

If two rows share the current maximum and one arrives after the run, it's skipped
permanently. An overlap fixes it, and with a `unique_key` the overlap is free —
[B6](../B-write-patterns/B6-watermark-filter.md).

## Did the script handle it?

Usually check three things in the original:

- **A wider range than the schedule needs** — `interval 7 day` on a daily job.
  That's a lookback, and it's deliberate even if undocumented.
- **`>=` rather than `>`** on the watermark. Also deliberate.
- **Neither.** The script has been losing late data, quietly. That's a finding for
  [A9](../A-assess/A9-correctness-baseline.md), and fixing it produces a
  legitimate diff you should predict —
  [H11](../H-verification/H11-differences-that-should-exist.md).

The third case is common, and it's why "the new model has more rows" during
verification is sometimes correct rather than a bug.

## Detect what you're still missing

Whatever window you choose, some data will arrive outside it. Measure it rather
than assuming zero:

```sql
select count(*) as arrived_too_late
from `raw.events`
where date_diff(date(_loaded_at), event_date, day) > 3
  and _loaded_at > timestamp_sub(current_timestamp(), interval 7 day);
```

Make it a `warn`-severity test. Then a change in upstream behaviour surfaces as a
warning instead of as quietly missing rows.

---

Previous: [G7 · Backfill via explicit partition ranges](G7-backfill-partition-ranges.md) ·
Next: [G9 · Selectors](G9-selectors.md)
