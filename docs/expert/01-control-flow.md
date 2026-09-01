# 1. Control flow and dispatch

`dbt-bigquery/.../materializations/incremental.sql`

## Materialization branch order

First match wins.

| # | Guard | Action |
| --- | --- | --- |
| 1 | `partition_by.copy_partitions` and strategy ∉ {`insert_overwrite`, `microbatch`} | `raise_compiler_error` |
| 2 | `existing_relation is none` | `bq_create_table_as(...)` |
| 3 | `existing_relation.is_view` | `drop_relation` then `bq_create_table_as` — non-atomic |
| 4 | `should_full_refresh()` | `drop_relation` **if** `not adapter.is_replaceable(...)`, then create |
| 5 | else | strategy dispatch |

Strategy validation (`dbt_bigquery_validate_get_incremental_strategy`) runs
before branch 1 — invalid strategy fails without executing SQL. Default is
`merge`. `microbatch` additionally triggers `bq_validate_microbatch_config`.

`is_replaceable` = `_partitions_match ∧ _clusters_match`. Returns `True` on
`NotFound` or falsy relation. `_partitions_match` compares field name and
`partitioning_type` case-insensitively for time partitioning, the range dict for
range partitioning — **`data_type` is not compared** for time partitioning.
Mismatch logs `Hard refreshing <relation> because it is not replaceable`.

## Temp relation condition

```jinja
{% if on_schema_change != 'ignore' or language == 'python' %}
```

`on_schema_change` defaults to `'ignore'` via `incremental_validate_on_schema_change`.
So the default SQL path sets `tmp_relation_exists = false` and passes
`compiled_code` down for inlining.

Downstream consumption of `tmp_relation_exists = false`:

| Path | Behaviour |
| --- | --- |
| `merge` | inline `( {{ sql }} )` as `USING`; builder never creates a temp relation |
| `insert_overwrite` static | inline `( {{ sql }} )` as `USING` |
| `insert_overwrite` static + `copy_partitions` | `create_tmp_relation_for_copy` statement |
| `insert_overwrite` dynamic | `bq_create_table_as` as step 1 of the script |
| `insert_overwrite` dynamic + `copy_partitions` | `create_tmp_relation_for_copy` statement |

Consequence: setting `on_schema_change` on a `merge` model changes the plan from
one statement to two, and fully materializes model output before the merge.

## Dispatch chain

```
bq_generate_incremental_build_sql
├── insert_overwrite → bq_generate_incremental_insert_overwrite_build_sql
│                      └── (partition_by none → compiler error)
│                          bq_insert_overwrite_sql
│                          ├── partitions non-empty → bq_static_insert_overwrite_sql
│                          │     ├── copy_partitions → bq_static_copy_partitions_insert_overwrite_sql
│                          │     └── else           → get_insert_overwrite_merge_sql(..., include_sql_header=not tmp_exists)
│                          └── else                 → bq_dynamic_insert_overwrite_sql
│                                ├── copy_partitions → bq_dynamic_copy_partitions_insert_overwrite_sql
│                                └── else            → get_insert_overwrite_merge_sql(...)
├── microbatch      → bq_generate_microbatch_build_sql
│                      └── bq_insert_overwrite_sql   ← identical entry point
└── merge (else)    → bq_generate_incremental_merge_build_sql
                       └── get_merge_sql → default__get_merge_sql
```

Static/dynamic selection is `partitions is not none and partitions != []`.

## dest_columns resolution

```
on_schema_change != 'ignore' → dispatch('process_schema_changes') → source_columns (temp relation)
                    'ignore' → returns {} → falsy → get_columns_in_relation(existing_relation)
```

Then, if `time_ingestion_partitioning`: `add_time_ingestion_partition_column`.

Under `ignore`, the column list comes from the **target**. Newly produced columns
are absent from the insert list and silently dropped.

## Post-main sequence

`run_hooks(post_hooks)` → `this.incorporate(type='table')` → `apply_grants`
(revoke gated on `should_revoke(existing_relation, full_refresh_mode)`) →
`persist_docs` → `drop_relation(tmp_relation)` if `tmp_relation_exists`.

Several strategy paths also emit their own `drop table if exists` inside the
generated script. Cleanup is duplicated, not owned.

## Python constraints

`supported_languages=['sql','python']`, with hard failures:

- `insert_overwrite` + python → compiler error
- `time_ingestion_partitioning` + python → compiler error
- python always creates a temp relation
- `_dbt_max_partition` gated on `language == 'sql'`

---

Next: [2. Generated SQL](02-generated-sql.md)
