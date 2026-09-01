# 4. Quick reference

## Strategy matrix

| | `merge` | `insert_overwrite` | `microbatch` |
| --- | --- | --- | --- |
| Default | ✅ | | |
| `partition_by` required | no | **yes** | **yes** |
| `unique_key` honoured | yes | no | no |
| No `unique_key` ⇒ | `on FALSE`, append | n/a | n/a |
| `copy_partitions` allowed | **no** | yes | yes |
| Python models | yes | **no** | n/a |
| Temp relation on default config | no | dynamic: yes / static: no | as `insert_overwrite` |
| Emitted SQL | `MERGE` on key | `MERGE ... not matched by source` | identical to `insert_overwrite` |
| Cost scales with | target size | partitions touched | partitions touched |

## Config → behaviour

| Config | Default | Effect worth remembering |
| --- | --- | --- |
| `incremental_strategy` | `merge` | Validated pre-execution; 3 valid values |
| `unique_key` | none | Absent ⇒ `on FALSE` ⇒ append. List ⇒ bare `=`, not null-safe |
| `partition_by` | none | Strings lowercased at parse. Drives predicate rendering |
| `partitions` | none | Non-empty ⇒ static path. Interpolated verbatim |
| `copy_partitions` | `false` | `WRITE_TRUNCATE` per partition; one API call each |
| `on_schema_change` | `ignore` | **Non-`ignore` forces temp relation.** `ignore` drops new columns |
| `incremental_predicates` / `predicates` | none | ANDed into `ON`. Narrow window ⇒ duplicates |
| `merge_update_columns` | all | xor `merge_exclude_columns`; both ⇒ error |
| `merge_exclude_columns` | none | Case-insensitive; rebuilds quoted list |
| `require_partition_filter` | `false` | Injects tautology. No pruning benefit |
| `time_ingestion_partitioning` | `false` | Rewrites SQL, drops partition column from output |
| `batch_size` | none | Must `==` `partition_by.granularity`, case-sensitively |
| `sql_header` | none | On static i/o, included only when `not tmp_relation_exists` |

## Compile-time errors

| Trigger | Message fragment |
| --- | --- |
| strategy ∉ {merge, insert_overwrite, microbatch} | *Invalid incremental strategy provided* |
| `copy_partitions` + `merge` | *requires the 'incremental_strategy' option to be set to 'insert_overwrite' or 'microbatch'* |
| `insert_overwrite` without `partition_by` | *requires the `partition_by` config* |
| `microbatch` without `partition_by` | *requires a `partition_by` config* |
| `granularity != batch_size` | *same granularity as its configured `batch_size`* |
| both merge column lists | *cannot specify merge_update_columns and merge_exclude_columns* |
| `insert_overwrite` + python | *not yet supported for python models* |
| ingestion-time + python | *Python models do not support ingestion time partitioning* |
| `on_schema_change='fail'` + drift | *source and target schemas ... are out of sync!* |
| malformed `partition_by` | *Could not parse partition config* / *Expected a dictionary with "field" and "data_type" keys* |

Runtime: `Undeclared query parameter` for `_dbt_max_partition` ⇒ literal
substring gate not satisfied.

## Silent-failure checklist

Ranked by how often it costs a day.

1. **Empty partition retained** — dynamic i/o, zero rows for a partition ⇒ no delete.
2. **New column dropped** — `on_schema_change='ignore'`, `dest_columns` from target.
3. **Composite-key duplicates** — nullable key column, bare `=`.
4. **Append-only merge** — no `unique_key`, `on FALSE`.
5. **Null-partition accumulation** — `IGNORE NULLS` excludes from replacement set.
6. **Predicate-window duplicates** — row older than `incremental_predicates` bound.
7. **Silent widening** — `can_expand_to` ⇒ no `alter` emitted.
8. **`interval: 1` pruning loss** — released 1.12.0 only.
9. **Static literals at wrong granularity** — compared against rendered expression.

## Source map

Relative to repo root of `dbt-labs/dbt-adapters`.

| Concern | Path |
| --- | --- |
| Materialization, strategy validation | `dbt-bigquery/src/dbt/include/bigquery/macros/materializations/incremental.sql` |
| insert_overwrite, copy_partitions | `.../materializations/incremental_strategy/insert_overwrite.sql` |
| merge override, unique-key match | `.../incremental_strategy/merge.sql` |
| microbatch validation + delegation | `.../incremental_strategy/microbatch.sql` |
| `_dbt_max_partition`, partition-filter tautology | `.../incremental_strategy/common.sql` |
| Ingestion-time rewrite | `.../incremental_strategy/time_ingestion_tables.sql` |
| BQ schema-change + STRUCT sync | `.../materializations/models/incremental/on_schema_change.sql` |
| Generic MERGE builders | `dbt-adapters/src/dbt/include/global_project/macros/materializations/models/incremental/merge.sql` |
| `get_quoted_csv`, `diff_columns`, update columns | `.../models/incremental/column_helpers.sql` |
| `is_incremental()` | `.../models/incremental/is_incremental.sql` |
| Strategy arg-dict dispatch (non-BQ path) | `.../models/incremental/strategies.sql` |
| `PartitionConfig` | `dbt-bigquery/src/dbt/adapters/bigquery/relation_configs/_partition.py` |
| `is_replaceable`, `copy_table`, behaviour flags | `dbt-bigquery/src/dbt/adapters/bigquery/impl.py` |

Pinned commit and package version: [repository README](../../README.md).

## Reading the other tracks

Mechanism and rationale: [balanced track](../balanced/01-how-the-materialization-runs.md).
Ground-up: [beginner track](../beginner/README.md).

---

Previous: [3. Semantics and edge cases](03-semantics.md) ·
Back to [the expert index](README.md)
