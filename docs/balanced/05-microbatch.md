# 5. The `microbatch` strategy

Source: `dbt-bigquery/src/dbt/include/bigquery/macros/materializations/incremental_strategy/microbatch.sql`

The adapter-side implementation is 28 lines, and most of it is an error message.
That is the main thing to understand about `microbatch` on BigQuery: **almost
none of it lives in the adapter.**

## What the adapter actually does

Two macros, in full.

`bq_validate_microbatch_config` runs at compile time, called from
`dbt_bigquery_validate_get_incremental_strategy` before any SQL is generated:

1. `partition_by` must be set, or *The 'microbatch' strategy requires a
   `partition_by` config.*
2. `partition_by.granularity` must equal `batch_size`, or an error naming both
   values.

`bq_generate_microbatch_build_sql` then does exactly one thing:

```jinja
{% set build_sql = bq_insert_overwrite_sql(
    tmp_relation, target_relation, sql, unique_key, partition_by, partitions,
    dest_columns, tmp_relation_exists, copy_partitions
) %}
```

That is the same entry point `insert_overwrite` uses. **The generated SQL is
identical.** Static vs dynamic selection, `copy_partitions`, the `MERGE` shape,
`_dbt_max_partition` — all exactly as described on
[page 4](04-insert-overwrite.md), with no microbatch-specific variation.

So everything on page 4 applies here, including
[the empty-partition trap](04-insert-overwrite.md#the-empty-partition-trap).
A batch that produces no rows does not clear the corresponding partition.

## The granularity check

The rule is strict equality between `partition_by.granularity` and `batch_size`:

```jinja
{% if config.get("partition_by").granularity != config.get('batch_size') %}
```

Note it reads the **raw** config dict, not the parsed `PartitionConfig`. The
parsed object lowercases its string values; the raw dict does not. It is also a
plain `!=`, so this comparison is case-sensitive. Keep both values lowercase and
identical.

The check also means `granularity` is effectively mandatory when you use
`microbatch`. It defaults to `day` in `PartitionConfig`, but that default is
applied during parsing — this validation reads the raw dict, where an omitted
key is `None` and will not equal your `batch_size`.

## What lives in dbt-core instead

Everything that makes `microbatch` *microbatch* is orchestration, and none of it
is in `dbt-adapters`. Searching the repository at the pinned commit for
`event_time`, `lookback`, or `__dbt_internal_microbatch` returns no hits in the
incremental macros; the only `batch_size` reference outside this file is an
unrelated seed helper.

That orchestration lives in `dbt-core`, which is now pinned alongside the adapter
— see [the repository README](../../README.md). Source:
`core/dbt/materializations/incremental/microbatch.py` and `core/dbt/task/run.py`.

### The configs

| Config | Default | Behaviour |
| --- | --- | --- |
| `batch_size` | none | Required. One of `hour`, `day`, `month`, `year` (the `BatchSize` enum). Missing ⇒ runtime error. |
| `begin` | `None` | Required on the first run, or when there's no checkpoint. Missing ⇒ *requires a 'begin' configuration*. |
| `lookback` | `1` | How many batches before the checkpoint to reprocess. |
| `event_time` | `None` | The column batches are cut on. |

All timestamps are coerced to **UTC** (`pytz.UTC`).

### How the window is computed

`build_end_time` takes `event_time_end` if given, else now, and rounds it **up**
to the batch boundary (`ceiling_timestamp`).

`build_start_time` has three cases, in order:

1. `event_time_start` given ⇒ use it, truncated down to the batch boundary.
2. First run, or no checkpoint ⇒ use `begin`, truncated.
3. Otherwise ⇒ `offset_timestamp(checkpoint, batch_size, -lookback)`.

Case 3 carries a subtlety worth knowing. If the checkpoint sits *exactly* on a
batch boundary, `lookback` is silently incremented by one:

```python
if checkpoint == MicrobatchBuilder.truncate_timestamp(checkpoint, batch_size):
    lookback += 1
```

The source explains why: a checkpoint on the line means the last batch *ends*
there, so the start has to come from the previous period. A run landing exactly
on midnight therefore reprocesses one batch more than `lookback` suggests.

### How batches are cut

`build_batches` walks from start to end, each batch running from one boundary to
the next, then overwrites the final batch's end with the **exact** `end` value
rather than a rounded one. So every batch is a whole period except the last,
which is truncated to the real end time.

### Hooks fire per model, not per batch

This one surprises people, and it's in `task/run.py`:

```python
# Only run pre_hook(s) for first batch
if batch_idx != 0:
    node_copy.config.pre_hook = []

# Only run post_hook(s) for last batch
if batch_idx != len(batches) - 1:
    node_copy.config.post_hook = []
```

A 400-batch backfill runs your `pre_hook` **once** and your `post_hook` **once**,
not 400 times each. If you were relying on a post-hook to fire per batch, it
won't — and if you were worried it would, it doesn't.

### Each batch's Jinja context

`build_jinja_context_for_batch` sets, for batches running incrementally:

```python
jinja_context["is_incremental"] = lambda: True
jinja_context["should_full_refresh"] = lambda: False
```

So inside a batch, `is_incremental()` is **forced true** and
`should_full_refresh()` **forced false**, regardless of what the adapter-side
macros would otherwise have computed.

## When it's worth it

Given the SQL is identical to `insert_overwrite`, the reason to choose
`microbatch` is never the generated statement. It is that dbt-core will split one
enormous run into many bounded ones, each retryable on its own.

Backfilling three years of daily data as a single `insert_overwrite` run means
one temp table holding three years of rows, one array of ~1,100 partitions, and
one `MERGE` — which may simply not complete. The same work as 1,100 batches will.

For a routine daily run touching one or two partitions, `microbatch` adds
orchestration overhead and buys nothing the simpler strategy doesn't already do.

---

Previous: [4. The `insert_overwrite` strategy](04-insert-overwrite.md) ·
Next: [6. `partition_by` in detail](06-partition-config.md)
