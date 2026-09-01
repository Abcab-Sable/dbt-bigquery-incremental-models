# 5. The three strategies

BigQuery accepts exactly three values for `incremental_strategy`: `merge`,
`insert_overwrite`, and `microbatch`. Anything else fails immediately with
*Invalid incremental strategy provided*. If you don't set it, you get `merge`.

Here's the honest summary before the detail:

| | Works on | Replaces | Needs `partition_by` |
| --- | --- | --- | --- |
| `merge` | individual rows | one row at a time | no |
| `insert_overwrite` | whole partitions | a whole day at a time | **yes** |
| `microbatch` | whole partitions, in batches | a whole day at a time | **yes** |

## `merge` — row by row

The default. You met it on [page 3](03-your-first-incremental-model.md).

For each incoming row, find the matching existing row by `unique_key`; update it
if found, insert it if not.

**Use it when** rows genuinely change in place and you have a real key — orders,
customers, subscriptions. Anything with a lifecycle.

**Its weakness is cost.** To find matches, BigQuery must search the target table.
Without partitioning and a bound, that's a full scan of your entire table every
single run. `merge` costs scale with **how big your table is**, not with how much
new data you have — precisely the problem we set out to solve.

You can bound it with `incremental_predicates`:

```sql
{{ config(
    materialized='incremental',
    unique_key='order_id',
    partition_by={'field': 'order_date', 'data_type': 'date'},
    incremental_predicates=[
        "DBT_INTERNAL_DEST.order_date >= date_sub(current_date(), interval 7 day)"
    ]
) }}
```

That restricts matching to the last 7 days of the target, so BigQuery prunes to
7 partitions instead of reading everything.

**Read that config as a statement about your data, not a speed setting.** You've
just told dbt that orders older than 7 days never change. If a 30-day-old order
does change, it won't match anything — and it'll be **inserted as a duplicate**.
Set the window to match reality.

## `insert_overwrite` — partition by partition

Instead of matching rows, throw away whole partitions and write fresh ones.

Concretely: "delete everything in the partitions I'm about to write, then write
them."

```sql
{{ config(
    materialized='incremental',
    incremental_strategy='insert_overwrite',
    partition_by={'field': 'event_date', 'data_type': 'date', 'granularity': 'day'}
) }}

select
    event_date,
    user_id,
    count(*) as events
from {{ source('raw', 'events') }}
{% if is_incremental() %}
  where event_date >= date_sub(current_date(), interval 3 day)
{% endif %}
group by 1, 2
```

Each run recomputes the last 3 days and replaces those 3 partitions wholesale.

**No `unique_key` needed.** The partition is the unit of replacement, so
individual row identity doesn't matter. That's a genuine simplification —
aggregations like the one above have no natural row key anyway.

**Cost scales with partitions touched**, not with table size. Three partitions
today, three tomorrow, regardless of whether the table holds one year or ten.

### What it actually runs

dbt generates a small script:

```sql
-- 1. put this run's output in a temp table
create or replace table tmp_events as ( <your select> );

-- 2. find which partitions that output covers
set (dbt_partitions_for_replacement) = (
    select as struct array_agg(distinct event_date IGNORE NULLS)
    from tmp_events
);

-- 3. delete those partitions from the real table, then insert
merge into events as DBT_INTERNAL_DEST
    using (select * from tmp_events) as DBT_INTERNAL_SOURCE
    on FALSE
when not matched by source
     and DBT_INTERNAL_DEST.event_date in unnest(dbt_partitions_for_replacement)
    then delete
when not matched then insert (...) values (...);

-- 4. tidy up
drop table if exists tmp_events;
```

Worth reading step 2 twice, because it's where the surprise lives.

**dbt works out which partitions to replace by looking at the rows your model
produced.** Not from your config, not from your `where` clause — from the actual
output.

So if your model produces no rows for a day, that day isn't in the list, and
**that day's existing data is left completely alone**. Your model said "this day
should be empty" and dbt heard nothing at all.

That's the empty-partition trap, and it gets
[a full walkthrough](06-when-things-go-wrong.md#the-partition-that-would-not-empty)
because it is the single most common way these models go quietly wrong.

### Naming the partitions yourself

You can take the guessing away by listing partitions explicitly:

```sql
{{ config(
    materialized='incremental',
    incremental_strategy='insert_overwrite',
    partition_by={'field': 'event_date', 'data_type': 'date'},
    partitions=['date_sub(current_date(), interval 1 day)', 'current_date()']
) }}
```

Now dbt overwrites yesterday and today **whether or not your model produced rows
for them**. Empty means empty. This is the fix for the trap above.

Two cautions. These values are pasted into the SQL literally, so they're
expressions — `current_date()` works, but a fixed date needs its own quotes
(`'2026-08-31'`). And never build this list from user input, since there's no
escaping whatsoever.

## `microbatch` — the same thing, in slices

Here is the fact that saves you a lot of confusion:

**On BigQuery, `microbatch` generates exactly the same SQL as
`insert_overwrite`.** The macro that builds it calls the `insert_overwrite`
builder and does nothing else.

The difference is that dbt splits one big run into many smaller runs, each
covering a slice of time, each executed and retried separately.

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

**`batch_size` must exactly equal `partition_by.granularity`** or dbt refuses to
compile, naming both values in the error. One batch, one partition.

**Use it for backfills.** Three years of daily data as one `insert_overwrite` run
means one temp table with three years of rows and a single enormous `MERGE`,
which may simply never finish. The same work as 1,100 daily batches will complete,
and a failure at batch 700 doesn't discard the first 699.

For a routine daily run touching one partition, microbatch adds orchestration
overhead and buys nothing.

Everything about the trap above applies here too — an empty batch doesn't clear
its partition.

> **Two things worth knowing**, both checked in dbt's source. `begin` is
> **required** — leave it out and the first run fails with *requires a 'begin'
> configuration*. And your `pre_hook` and `post_hook` run **once for the whole
> model, not once per batch**: the pre-hook fires on the first batch only, the
> post-hook on the last. A 400-batch backfill runs each of them once.
>
> More detail on the [balanced track](../balanced/05-microbatch.md#what-lives-in-dbt-core-instead).

## The hidden setting that changes everything

One config affects all three strategies in a way nobody expects.

`on_schema_change` controls what happens when your model starts producing
different columns than the table has. It defaults to `ignore`.

The obvious effect: with `ignore`, a **new column you add to your model will not
appear in the table**. No error. dbt takes the column list from the existing
table and quietly drops anything extra. This is far and away the most common
"why isn't my change showing up" question.

Fix it by asking dbt to sync:

| Value | What happens |
| --- | --- |
| `ignore` (default) | New columns silently discarded. |
| `fail` | Run stops with an error listing the differences. |
| `append_new_columns` | New columns added to the table. Nothing removed. |
| `sync_all_columns` | Added, removed, **and** retyped to match your model. |

`sync_all_columns` will **drop** columns your model no longer produces. That's
data loss, on purpose. Check the "Columns removed" line dbt logs.

The non-obvious effect: **changing this setting changes how your model runs.**
With `ignore`, dbt inlines your query directly into the `MERGE` — one statement.
With anything else, dbt must build a temp table first to compare columns — two
statements. Your query now fully materialises before the merge starts.

So `on_schema_change` is quietly a performance setting as well as a
schema setting. `append_new_columns` is usually the right choice anyway; just
know you changed more than you thought.

## Choosing

A rough decision path:

**Do rows change after they're first written?**
Yes, and you have a real key → **`merge`**, with `incremental_predicates` to bound
it and partitioning to make the bound work.

**Is your data event-shaped — one row per thing that happened, keyed on a date?**
→ **`insert_overwrite`**, and use explicit `partitions` if any period could
legitimately become empty.

**Backfilling a long history, or single runs too big to finish?**
→ **`microbatch`**.

**Not sure?** `merge` with a `unique_key` is the forgiving default. It's slower
but harder to get silently wrong, and correctness beats cost while you're
learning.

---

Previous: [4. Partitions, explained properly](04-partitioning-explained.md) ·
Next: [6. When things go wrong](06-when-things-go-wrong.md)
