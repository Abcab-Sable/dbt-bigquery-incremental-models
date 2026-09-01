# dbt incremental models on BigQuery — a source-derived reference

The dbt documentation covers incremental models at a level that works until it
doesn't. This reference fills the gap by describing what the adapter macros
*actually do*: which branch runs, what SQL is generated, and where the behaviour
surprises people.

Every claim here was written by reading the macro and Python source in
[`dbt-labs/dbt-adapters`](https://github.com/dbt-labs/dbt-adapters), not from the
published docs. Where the source and the intuition disagree, this reference
follows the source.

## Pick your track

The same material is written three times, for three different readers. They cover
identical behaviour and identical findings — only the depth of explanation
differs.

| Track | For | Length |
| --- | --- | --- |
| 🌱 **[Beginner](docs/beginner/README.md)** | No prior knowledge assumed. Explains what a partition is, what a `MERGE` does, what dbt is for. | 7 pages, long |
| ⚖️ **[Balanced](docs/balanced/README.md)** | You know dbt and BigQuery. Explains mechanisms, not just behaviours. | 8 pages |
| ⚡ **[Expert](docs/expert/README.md)** | You know this already and need the specifics. Tables, emitted SQL, edge cases. | 4 pages, terse |

**Not sure?** Start with [balanced](docs/balanced/README.md). Drop to
[beginner](docs/beginner/README.md) if a page assumes something you don't have;
jump to [expert](docs/expert/README.md) if you're skimming for a detail you
half-remember.

## The findings, if you only read one thing

- **Dynamic `insert_overwrite` never empties a partition.** The replacement set is
  derived from the rows your model produced, so a partition that should become
  empty keeps its old contents. The run succeeds.
  → [beginner](docs/beginner/06-when-things-go-wrong.md#the-partition-that-would-not-empty)
  · [balanced](docs/balanced/04-insert-overwrite.md#the-empty-partition-trap)
  · [expert](docs/expert/02-generated-sql.md#insert_overwrite--dynamic)
- **`on_schema_change` is secretly a performance setting.** Anything other than
  `ignore` forces a temp table, turning a one-statement `merge` into two.
  → [balanced](docs/balanced/01-how-the-materialization-runs.md#when-a-temp-table-is-created-and-when-it-isnt)
- **Composite `unique_key` is not null-safe**, even with
  `enable_truthy_nulls_equals_macro` enabled — the list branch never reaches the
  null-safe macro. → [expert](docs/expert/03-semantics.md#unique_key-three-branches-two-behaviours)
- **`merge` without a `unique_key` is append-only.** The merge condition becomes
  `on FALSE`. → [balanced](docs/balanced/02-choosing-a-strategy.md#merge-without-a-unique_key-is-append-only)
- **`microbatch` emits identical SQL to `insert_overwrite`** on BigQuery. The
  batching is entirely dbt-core. → [balanced](docs/balanced/05-microbatch.md)
- **`require_partition_filter` gives you no pruning.** dbt satisfies it with a
  tautology. → [balanced](docs/balanced/03-merge.md#the-require_partition_filter-predicate)

## Source of truth

| Component | Where the logic lives |
| --- | --- |
| BigQuery `incremental` materialization | `dbt-bigquery/src/dbt/include/bigquery/macros/materializations/incremental.sql` |
| Strategy macros | `dbt-bigquery/src/dbt/include/bigquery/macros/materializations/incremental_strategy/` |
| Generic `MERGE` builders | `dbt-adapters/src/dbt/include/global_project/macros/materializations/models/incremental/merge.sql` |
| `partition_by` parsing and rendering | `dbt-bigquery/src/dbt/adapters/bigquery/relation_configs/_partition.py` |
| Full-refresh replaceability checks | `dbt-bigquery/src/dbt/adapters/bigquery/impl.py` |
| Batch orchestration for `microbatch` | `dbt-core` — **not** in this repo, see [the microbatch page](docs/balanced/05-microbatch.md) |

A per-concern file map is in the
[expert quick reference](docs/expert/04-quick-reference.md#source-map).

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
  Released 1.12.0 applies the normalisation unconditionally, which defeats
  partition elimination. Detailed in
  [balanced](docs/balanced/06-partition-config.md#int64-range-partitions-get-boundary-normalisation)
  and [expert](docs/expert/03-semantics.md#released-vs-main-delta).

Unless a page says otherwise, it describes **released 1.12.0** — the version you
get from `pip install dbt-bigquery`.

## Scope

BigQuery only. Other adapters implement the same strategy names with different
SQL, and the differences are not cosmetic — don't carry these conclusions across.

The `microbatch` batching machinery (`event_time`, `begin`, `lookback`, batch
splitting) lives in dbt-core and was **not** read. It is flagged as unverified
wherever it comes up, rather than described from memory.

This is plain markdown with no build step. Read it on github.com, or clone it.

## Licence

Documentation, so [CC BY 4.0](LICENSE) — reuse it anywhere with attribution.
