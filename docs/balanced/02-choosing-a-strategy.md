# 2. Choosing a strategy

Source: `dbt-bigquery/src/dbt/include/bigquery/macros/materializations/incremental.sql`,
dispatching into `incremental_strategy/`.

Three strategies are valid on BigQuery: `merge` (the default), `insert_overwrite`,
and `microbatch`. `bq_generate_incremental_build_sql` routes to one of three
builders — and one of those routes is not what most people expect.

## The routing, exactly as written

```jinja
{% if strategy == 'insert_overwrite' %}
    bq_generate_incremental_insert_overwrite_build_sql(...)
{% elif strategy == 'microbatch' %}
    bq_generate_microbatch_build_sql(...)
{% else %} {# strategy == 'merge' #}
    bq_generate_incremental_merge_build_sql(...)
{% endif %}
```

Follow `bq_generate_microbatch_build_sql` and you find it does nothing but call
`bq_insert_overwrite_sql` — the same function `insert_overwrite` uses.

**`microbatch` on BigQuery is `insert_overwrite`.** The difference is entirely in
validation (page 5) and in how dbt-core slices the run into batches. The
generated SQL comes from the same macro. If you understand `insert_overwrite`,
you understand what `microbatch` emits.

## What actually distinguishes them

| | `merge` | `insert_overwrite` | `microbatch` |
| --- | --- | --- | --- |
| Requires `partition_by` | No | **Yes** — compiler error without it | **Yes** — compiler error without it |
| Requires `unique_key` | No (see below) | No — ignored in the dynamic path | No |
| Unit of replacement | Row | Partition | Partition |
| Can delete rows that vanished from source | Only within a matched key | Yes, within touched partitions | Yes, within touched batches |
| Works with `copy_partitions` | **No** — compiler error | Yes | Yes |
| Python models | Yes | **No** — compiler error | Not applicable |
| Temp table on default config | No | Yes (dynamic) / No (static) | Same as `insert_overwrite` |

## Picking one

**Use `merge` when rows change in place and you have a real key.** It's the
default, it doesn't require partitioning, and it handles late-arriving updates to
old rows without you having to reason about which partitions those rows landed
in.

Its cost profile is the catch: the `MERGE` must locate matching rows in the
target. On a large unpartitioned table that's a full scan every run. `merge`
scales with *target* size, not with how much new data you have.

**Use `insert_overwrite` when your data is partitioned and you rebuild whole
partitions.** Cost scales with the number of partitions you touch, not with table
size. This is the right default for event and log data keyed on a date.

The trade-off is that it is partition-shaped, not row-shaped. A row that moves
from one partition to another leaves a copy behind in the old partition unless
that partition is also rebuilt. Correctness depends on your model emitting
*complete* partitions — see [the empty-partition trap](04-insert-overwrite.md#the-empty-partition-trap).

**Use `microbatch` when a single run would be too big to execute as one
statement.** Backfills over long histories are the motivating case. You get
per-batch retry and parallelism from dbt-core, at the price of more statements.

## `merge` without a `unique_key` is append-only

Worth stating on its own, because it's silent. In `default__get_merge_sql`:

```jinja
{% if unique_key %}
    ... build the match predicate ...
{% else %}
    {% do predicates.append('FALSE') %}
{% endif %}
```

With no `unique_key`, the `MERGE` runs `on FALSE`. Nothing ever matches, the
`when matched` clause is omitted entirely, and every source row is inserted.

That is an append. It will happily accumulate duplicates run after run, and dbt
will not warn you. If you meant "insert only new rows", you need a `unique_key`
or a filter in your model on `is_incremental()`.

## A note on cost intuition

The instinct that `insert_overwrite` is cheaper than `merge` is usually right,
but the mechanism isn't the one people describe. Both strategies emit a `MERGE`
statement. `insert_overwrite` is cheaper because its predicate restricts the
target side to a bounded set of partitions, which BigQuery can prune to — not
because it avoids `MERGE`.

That's also why the partition predicate has to be *prunable*. When it isn't, the
cost advantage evaporates while the SQL still looks correct. See
[page 6](06-partition-config.md) on `render_wrapped`, and the
`require_partition_filter` interaction on [page 8](08-gotchas.md).

---

Previous: [1. How the materialization runs](01-how-the-materialization-runs.md) ·
Next: [3. The `merge` strategy](03-merge.md)
