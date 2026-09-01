# 3. The `merge` strategy

Sources:
`dbt-bigquery/src/dbt/include/bigquery/macros/materializations/incremental_strategy/merge.sql`
and `dbt-adapters/src/dbt/include/global_project/macros/materializations/models/incremental/merge.sql`.

`merge` is the default strategy. BigQuery's builder,
`bq_generate_incremental_merge_build_sql`, is thin: it assembles the source
subquery, collects predicates, and hands off to the generic `get_merge_sql`.

## The generated statement

From `default__get_merge_sql`, the shape is:

```sql
merge into <target> as DBT_INTERNAL_DEST
    using ( <your model sql, or select * from tmp> ) as DBT_INTERNAL_SOURCE
    on (<predicate>) and (<predicate>) ...

when matched then update set
    <col> = DBT_INTERNAL_SOURCE.<col>, ...

when not matched then insert
    (<cols>)
values
    (<cols>)
```

The `when matched then update set` block is **omitted entirely** when there is no
`unique_key`. The `when not matched then insert` block is always present.

## Where the source comes from

```jinja
{%- if tmp_relation_exists -%}
    ( select * from {{ tmp_relation }} )
{%- else -%}
    ( {{ sql }} )
{%- endif -%}
```

On a default config (`on_schema_change = 'ignore'`, SQL model), `tmp_relation_exists`
is false, so **your model's compiled SQL is inlined into the `MERGE`**. There is
no intermediate table. See [page 1](01-how-the-materialization-runs.md#when-a-temp-table-is-created-and-when-it-isnt)
for when this flips.

## `unique_key` semantics

Three distinct cases, and they do not behave the same way.

### No `unique_key`

`predicates` gets `'FALSE'`. The merge condition is `on (FALSE)`, nothing matches,
everything inserts. This is an append — covered on [page 2](02-choosing-a-strategy.md#merge-without-a-unique_key-is-append-only).

### A single `unique_key` string

Builds `DBT_INTERNAL_SOURCE.<key>` and `DBT_INTERNAL_DEST.<key>` and passes them
through `get_merge_unique_key_match`, which BigQuery overrides.

### A list `unique_key`

This branch is different, and the difference is easy to miss:

```jinja
{% for key in unique_key %}
    {% set this_key_match %}
        DBT_INTERNAL_SOURCE.{{ key }} = DBT_INTERNAL_DEST.{{ key }}
    {% endset %}
    {% do predicates.append(this_key_match) %}
{% endfor %}
```

Each key is compared with a **plain `=`**, appended as a separate predicate and
joined with `AND`. This branch never calls `get_merge_unique_key_match`, so the
null-safe comparison described below **does not apply to composite keys**.

If any component of a composite key is `NULL`, `=` yields `NULL`, the row never
matches, and it is inserted as a duplicate on every run. Composite keys with
nullable components are a duplicate-generating machine, and the flag that fixes
it for single keys does not reach here.

## Null-safe equality and why BigQuery overrides it

The generic `default__get_merge_unique_key_match` delegates to the `equals`
macro. Under the `enable_truthy_nulls_equals_macro` behaviour flag, `equals`
emits `IS NOT DISTINCT FROM`, so `NULL = NULL` matches.

BigQuery overrides this in `bigquery__get_merge_unique_key_match`:

```jinja
{%- if adapter.behavior.enable_truthy_nulls_equals_macro.no_warn -%}
    (({{ source_unique_key }} is null and {{ target_unique_key }} is null)
     or ({{ source_unique_key }} = {{ target_unique_key }}))
{%- else -%}
    ({{ source_unique_key }} = {{ target_unique_key }})
{%- endif %}
```

Semantically identical to `IS NOT DISTINCT FROM`, but written out longhand. The
reason is given in the source comment: inside a `MERGE` on a table with
`require_partition_filter=True`, BigQuery's partition-pruning analyser stops
recognising the auxiliary `(field is null or field is not null)` predicate as a
valid partition filter when `IS NOT DISTINCT FROM` is present, and **the merge
fails at runtime**. The longhand form keeps pruning working.

This is a good example of why reading the source pays off: the flag is generic,
the failure is BigQuery-specific, and the workaround is invisible from the config.

## Controlling which columns update

`get_merge_update_columns` reads two configs:

- `merge_update_columns` — allow-list; only these are updated.
- `merge_exclude_columns` — deny-list; everything except these is updated.

Setting **both raises a compiler error**: *Model cannot specify
merge_update_columns and merge_exclude_columns.*

With neither set, every destination column is updated. Note the asymmetry in the
source: `merge_update_columns` is used **as given**, while `merge_exclude_columns`
is matched case-insensitively (`column.column | lower not in merge_exclude_columns | map("lower")`)
and rebuilds the list from `dest_columns`. So the exclude path yields properly
quoted identifiers; the update path uses your literal strings. If your column
names need quoting, the allow-list is the riskier of the two.

## `incremental_predicates`

Read as `config.get('predicates') or config.get('incremental_predicates')` — the
short name wins. Both are accepted; `get_merge_sql` also still honours a
`predicates` kwarg for backwards compatibility.

Whatever you supply is appended to the `ON` clause with `AND`. Use it to bound
the target side so the merge can prune:

```sql
{{ config(
    materialized='incremental',
    unique_key='id',
    incremental_predicates=["DBT_INTERNAL_DEST.event_date >= date_sub(current_date(), interval 7 day)"]
) }}
```

That restricts matching to the last 7 days of the target. Rows older than that
will no longer match and will be **inserted as duplicates** if they appear in
your source. The predicate is a correctness statement about your data, not just
a cost knob.

## The `require_partition_filter` predicate

`predicate_for_avoid_require_partition_filter` (in `common.sql`) injects an
extra predicate when the model is partitioned **and** `require_partition_filter`
is set:

```sql
(
    `DBT_INTERNAL_DEST`.`<partition_field>` is null
    or `DBT_INTERNAL_DEST`.`<partition_field>` is not null
)
```

A tautology, present purely to satisfy BigQuery's requirement that a partition
filter appear. It satisfies the check without restricting anything — which means
it does **not** give you pruning. If you set `require_partition_filter` expecting
it to force cheap merges, it won't; add a real `incremental_predicates` bound.

---

Previous: [2. Choosing a strategy](02-choosing-a-strategy.md) ·
Next: [4. The `insert_overwrite` strategy](04-insert-overwrite.md)
