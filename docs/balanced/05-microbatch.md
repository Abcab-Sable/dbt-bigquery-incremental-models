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

Everything that makes `microbatch` *microbatch* is orchestration, and it is not
in `dbt-adapters`. Searching the repository at the pinned commit for
`event_time`, `lookback`, or `__dbt_internal_microbatch` returns no hits in the
incremental macros. The only `batch_size` reference outside this file is an
unrelated seed helper.

So the following are **dbt-core** behaviour, and this reference does not document
them — it hasn't read that source:

- `event_time`, `begin`, `lookback` configs
- splitting a run into batches and computing each batch's time window
- injecting the time filter into your model
- per-batch execution, retry, and parallelism
- `--event-time-start` / `--event-time-end`

If you need those pinned to a version the way this reference pins the adapter,
read them in `dbt-labs/dbt-core` and record the commit alongside this one.

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
