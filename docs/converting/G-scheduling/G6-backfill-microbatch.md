# G6 · Backfill via microbatch

> **Part G — Scheduling, parameters, backfills** · Sourcing: `CORE✓`
> **The question:** the full refresh won't finish. What now?

Split it into batches and let dbt drive them. This is what `microbatch` is for —
not routine daily runs, but backfills too big to execute as one statement.

## The config

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

```bash
dbt run --select daily_events --event-time-start 2024-01-01 --event-time-end 2024-06-30
```

`batch_size` must **exactly equal** `partition_by.granularity`, or compilation
fails naming both values. The check reads the raw config dict and is
case-sensitive.

## What dbt does with it

From dbt-core's `MicrobatchBuilder`, all verified:

- **`begin` is required.** Missing on a first run ⇒ *Microbatch model ... requires
  a 'begin' configuration*.
- **`lookback` defaults to 1** — how many batches before the checkpoint to
  reprocess.
- Everything is coerced to **UTC**.
- End time is rounded **up** to a batch boundary; start is truncated **down**.
- The **final batch ends at the exact end value**, not a boundary.
- Each batch's Jinja context forces `is_incremental()` true and
  `should_full_refresh()` false.

Full detail in [the microbatch page](../../balanced/05-microbatch.md).

## The boundary subtlety worth knowing

If the checkpoint sits exactly on a batch boundary, `lookback` is silently
incremented:

```python
if checkpoint == MicrobatchBuilder.truncate_timestamp(checkpoint, batch_size):
    lookback += 1
```

A run landing precisely on midnight reprocesses one batch more than configured.
Harmless with idempotent batches; confusing if you're counting.

## Hooks fire once, not per batch

```python
# Only run pre_hook(s) for first batch
if batch_idx != 0:
    node_copy.config.pre_hook = []

# Only run post_hook(s) for last batch
if batch_idx != len(batches) - 1:
    node_copy.config.post_hook = []
```

A 400-batch backfill runs each hook **once**. And a post-hook signals "the final
batch completed", not "all batches succeeded" — don't use one as a backfill
completion signal ([F16](../F-hooks/F16-hooks-and-failure.md)).

## The generated SQL is `insert_overwrite`

On BigQuery, `bq_generate_microbatch_build_sql` delegates straight to
`bq_insert_overwrite_sql`. **Identical SQL.** Which means the
[empty-partition trap](../B-write-patterns/B14-when-the-range-can-empty.md)
applies per batch: a batch producing no rows doesn't clear its partition.

For a backfill over a period with genuinely quiet days, that matters. Use the
static `partitions` form if the batches must clear regardless.

## Why not just loop in bash

You could:

```bash
for d in $(seq ...); do dbt run --vars "{run_date: $d}"; done
```

Microbatch is better because dbt handles the boundary arithmetic (including the
lookback subtlety above), runs batches with proper concurrency, and records
per-batch results so a failure at batch 200 doesn't discard 199 successes.

`RunStatus` even has a `PartialSuccess` value for exactly this case.

## When not to use it

For a routine daily run touching one partition, microbatch adds orchestration
overhead and buys nothing plain `insert_overwrite` doesn't already do. It's a
backfill tool that happens to also work daily — not the other way round.

---

Previous: [G5 · Backfill via `--full-refresh`](G5-backfill-full-refresh.md) ·
Next: [G7 · Backfill via explicit partition ranges](G7-backfill-partition-ranges.md)
