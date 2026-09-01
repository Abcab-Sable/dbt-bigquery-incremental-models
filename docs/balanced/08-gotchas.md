# 8. Gotchas

The behaviours that cost people a day. Each links back to the page with the
mechanism.

## Data correctness

**Dynamic `insert_overwrite` never empties a partition.**
If your model produces zero rows for a partition, the old rows stay. Deletions do
not propagate, and the run succeeds. →
[The empty-partition trap](04-insert-overwrite.md#the-empty-partition-trap)

**`merge` without a `unique_key` is an append.**
The merge condition becomes `on FALSE`. Every row inserts, duplicates accumulate,
nothing warns you. →
[merge without a unique_key](02-choosing-a-strategy.md#merge-without-a-unique_key-is-append-only)

**Composite `unique_key` is not null-safe, even with the flag set.**
The list branch emits plain `=` per key and never calls
`get_merge_unique_key_match`. A `NULL` in any component means the row never
matches and is re-inserted every run. →
[unique_key semantics](03-merge.md#unique_key-semantics)

**Rows with a `NULL` partition value accumulate forever.**
`array_agg(distinct ... IGNORE NULLS)` excludes them from the replacement array,
so they insert but never replace. →
[Dynamic partitions](04-insert-overwrite.md#dynamic-partitions-the-common-case)

**`incremental_predicates` is a correctness statement, not just a cost knob.**
Bounding the target to the last 7 days means rows older than that stop matching
and get inserted as duplicates. →
[incremental_predicates](03-merge.md#incremental_predicates)

**`sync_all_columns` drops columns.**
By design. Check the "Columns removed" line in your run log. →
[The four values](07-schema-changes.md#the-four-values)

**Under `on_schema_change: ignore`, new columns are silently discarded.**
`dest_columns` comes from the existing target, so a newly produced column isn't
in the insert list. No error, no warning. →
[Return value](07-schema-changes.md#return-value)

**Static `partitions` values must be at partition granularity.**
They're compared against the *rendered* destination expression. With monthly
partitioning, `'2026-08-15'` never matches `date_trunc(dest, month)`. →
[Where each rendering is used](06-partition-config.md#where-each-rendering-is-used)

## Cost and performance

**`on_schema_change` changes your execution plan.**
Anything other than `ignore` forces a temp table, turning a one-statement `merge`
into a `CREATE` plus a `MERGE`. →
[When a temp table is created](01-how-the-materialization-runs.md#when-a-temp-table-is-created-and-when-it-isnt)

**`require_partition_filter` does not give you pruning.**
dbt injects `(field is null or field is not null)` — a tautology that satisfies
BigQuery's check without restricting anything. Add real
`incremental_predicates` if you want the merge bounded. →
[The require_partition_filter predicate](03-merge.md#the-require_partition_filter-predicate)

**`merge` cost scales with target size, not with new data.**
The merge locates matching rows in the target. On a large unpartitioned table
that's a full scan every run. →
[Picking one](02-choosing-a-strategy.md#picking-one)

**`int64` range partitioning with `interval: 1` blocks partition elimination on
released 1.12.0.**
The `MOD` normalisation is applied unconditionally there; `main` added a guard
that returns the raw column. →
[int64 range partitions](06-partition-config.md#int64-range-partitions-get-boundary-normalisation)

**`copy_partitions` loops in Python, one copy job per partition.**
Cheap per partition, but a run touching hundreds spends real time in that loop. →
[copy_partitions](04-insert-overwrite.md#copy_partitions)

## Runtime failures

**`_dbt_max_partition` is detected by literal substring match.**
`{%- if '_dbt_max_partition' in compiled_code and language == 'sql' -%}`. Build
the name dynamically and it is never declared — you get `Undeclared query
parameter`. Mention it in a comment and you get a spurious declaration. Python
models never get it. →
[_dbt_max_partition](04-insert-overwrite.md#_dbt_max_partition)

**`IS NOT DISTINCT FROM` breaks merges on tables with `require_partition_filter`.**
BigQuery's pruning analyser stops recognising the auxiliary predicate as a valid
partition filter and the merge fails at runtime. This is why BigQuery overrides
`get_merge_unique_key_match` to write the comparison longhand. →
[Null-safe equality](03-merge.md#null-safe-equality-and-why-bigquery-overrides-it)

**Changing `partition_by` or `cluster_by` turns `--full-refresh` into a drop.**
`is_replaceable` returns false, dbt logs `Hard refreshing ... because it is not
replaceable`, and issues a `DROP` before the `CREATE`. →
[Full refresh can silently become a drop](01-how-the-materialization-runs.md#full-refresh-can-silently-become-a-drop)

**Switching a model from `view` to `incremental` has a gap.**
There's no atomic view-to-table replacement on BigQuery, so dbt drops the view
first. The relation does not exist in between. →
[The branch order](01-how-the-materialization-runs.md#the-branch-order)

**Ingestion-time partitioning renames your column.**
The model SQL is rewritten to `select TIMESTAMP(field) as _PARTITIONTIME, *
EXCEPT(field)`. Downstream models can't select the original name. →
[Ingestion-time partitioning](06-partition-config.md#ingestion-time-partitioning)

## Compile-time errors

These fail before any SQL runs, which is the good case.

| Config | Error |
| --- | --- |
| `incremental_strategy` not in `merge`/`insert_overwrite`/`microbatch` | *Invalid incremental strategy provided* |
| `copy_partitions` with `merge` | *requires the 'incremental_strategy' option to be set to 'insert_overwrite' or 'microbatch'* |
| `insert_overwrite` without `partition_by` | *requires the `partition_by` config* |
| `microbatch` without `partition_by` | *requires a `partition_by` config* |
| `microbatch` where `granularity != batch_size` | error naming both values |
| Both `merge_update_columns` and `merge_exclude_columns` | *cannot specify ... Please update model to use only one config* |
| `insert_overwrite` on a Python model | *not yet supported for python models* |
| `time_ingestion_partitioning` on a Python model | *Python models do not support ingestion time partitioning* |

## Things that are not gotchas

Worth stating, since they get reported as bugs.

**`microbatch` generating `insert_overwrite` SQL is intentional.** The adapter
macro delegates directly to `bq_insert_overwrite_sql`. The batching happens in
dbt-core. → [page 5](05-microbatch.md)

**Both `predicates` and `incremental_predicates` work.** Read as
`config.get('predicates') or config.get('incremental_predicates')`. The short
name wins if both are set.

**Widening a column type produces no `alter`.** `diff_column_data_types` skips
columns where `can_expand_to` is true. → [What counts as a change](07-schema-changes.md#what-counts-as-a-change)

---

Previous: [7. Schema changes](07-schema-changes.md) ·
Back to [README](../../README.md)
