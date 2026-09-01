# dbt incremental models on BigQuery — a source-derived reference

The dbt documentation covers incremental models at a level that works until it
doesn't. This reference fills the gap by describing what the adapter macros
*actually do*: which branch runs, what SQL is generated, and where the behaviour
surprises people.

Every claim here was written by reading the macro and Python source in
[`dbt-labs/dbt-adapters`](https://github.com/dbt-labs/dbt-adapters), not from the
published docs. Where the source and the intuition disagree, this reference
follows the source.

## Start here

| Page | What it covers |
| --- | --- |
| [1. How the materialization runs](docs/01-how-the-materialization-runs.md) | The branch order in the BigQuery `incremental` materialization, and when a temp table is (and isn't) created |
| [2. Choosing a strategy](docs/02-choosing-a-strategy.md) | `merge` vs `insert_overwrite` vs `microbatch`, and what each one costs you |
| [3. The `merge` strategy](docs/03-merge.md) | The generated `MERGE`, `unique_key` semantics, `merge_update_columns`, null handling |
| [4. The `insert_overwrite` strategy](docs/04-insert-overwrite.md) | Static vs dynamic partitions, `copy_partitions`, and the empty-partition trap |
| [5. The `microbatch` strategy](docs/05-microbatch.md) | What the adapter validates, and what it delegates to dbt-core |
| [6. `partition_by` in detail](docs/06-partition-config.md) | How the config is parsed and rendered into predicates |
| [7. Schema changes](docs/07-schema-changes.md) | `on_schema_change`, and BigQuery's `STRUCT` synchronisation |
| [8. Gotchas](docs/08-gotchas.md) | The behaviours that cost people a day |

## Source of truth

| Component | Where the logic lives |
| --- | --- |
| BigQuery `incremental` materialization | `dbt-bigquery/src/dbt/include/bigquery/macros/materializations/incremental.sql` |
| Strategy macros | `dbt-bigquery/src/dbt/include/bigquery/macros/materializations/incremental_strategy/` |
| Generic `MERGE` builders | `dbt-adapters/src/dbt/include/global_project/macros/materializations/models/incremental/merge.sql` |
| `partition_by` parsing and rendering | `dbt-bigquery/src/dbt/adapters/bigquery/relation_configs/_partition.py` |
| Full-refresh replaceability checks | `dbt-bigquery/src/dbt/adapters/bigquery/impl.py` |
| Batch orchestration for `microbatch` | `dbt-core` — **not** in this repo, see [page 5](docs/05-microbatch.md) |

**Pinned to these exact versions, read on 2026-09-01:**

- **Repository:** `dbt-labs/dbt-adapters` at commit
  [`e7553c7fa6cdc82f1455be2035e4e948e7540792`](https://github.com/dbt-labs/dbt-adapters/commit/e7553c7fa6cdc82f1455be2035e4e948e7540792)
  (`main`, committed 2026-08-31)
- **Released package:** `dbt-bigquery` **1.12.0** on
  [PyPI](https://pypi.org/project/dbt-bigquery/1.12.0/)

Those two are not the same tree — `main` reports `1.12.0rc1` in its
`__version__.py`, while PyPI ships `1.12.0`. Both were checked:

- **All incremental macro files are byte-identical** between released 1.12.0 and
  `main` at the pinned commit. Everything this reference says about generated SQL
  applies to both.
- **`_partition.py` differs in one place.** `main` adds a guard so that an
  `int64` range partition with `interval: 1` skips the `MOD` normalisation.
  Released 1.12.0 applies the normalisation unconditionally. This is called out
  where it matters, in [page 6](docs/06-partition-config.md).

Unless a page says otherwise, it describes **released 1.12.0** — the version you
get from `pip install dbt-bigquery`.

## Scope

BigQuery only. Other adapters implement the same strategy names with different
SQL, and the differences are not cosmetic — don't carry these conclusions across.

This is plain markdown with no build step. Read it on github.com, or clone it.

## Licence

Documentation, so [CC BY 4.0](LICENSE) — reuse it anywhere with attribution.
