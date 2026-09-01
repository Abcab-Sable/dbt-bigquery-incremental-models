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

### 🔧 Converting existing scripts into dbt — *in progress, 63 of 138*

A fourth track, on turning scheduled queries, stored procedures and `bq`
pipelines into dbt models: archetype-by-archetype translation, hooks in depth,
backfills, and proving the conversion matches what it replaced.

Decomposed into **138 units across eleven parts**, sequenced into
[five delivery waves](docs/converting/BACKLOG.md#delivery-waves). **Waves 1–3 are complete**: assessment, every write-pattern archetype,
statement-level translation, hooks in full, and verification in full. Start at
[one statement per model](docs/converting/E-translation/E1-one-statement-per-model.md),
or see [the track index](docs/converting/README.md).

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
- **`microbatch` emits identical SQL to `insert_overwrite`** on BigQuery, and its
  hooks fire **once per model, not once per batch** — `pre_hook` on the first
  batch only, `post_hook` on the last. → [balanced](docs/balanced/05-microbatch.md)
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
| Batch orchestration for `microbatch` | `dbt-core`, `core/dbt/materializations/incremental/microbatch.py` |
| Hook execution and the `transaction` default | `dbt-core`, `core/dbt/artifacts/resources/v1/config.py` and `core/dbt/task/run.py` |

A per-concern file map is in the
[expert quick reference](docs/expert/04-quick-reference.md#source-map).

**Pinned to these exact versions, read on 2026-09-01:**

| Repository | Ref | Commit | Released package |
| --- | --- | --- | --- |
| `dbt-labs/dbt-adapters` | `main` | [`e7553c7`](https://github.com/dbt-labs/dbt-adapters/commit/e7553c7fa6cdc82f1455be2035e4e948e7540792) (2026-08-31) | `dbt-bigquery` [1.12.0](https://pypi.org/project/dbt-bigquery/1.12.0/) |
| `dbt-labs/dbt-core` | **`1.latest`** | [`300e80c`](https://github.com/dbt-labs/dbt-core/commit/300e80c7a24a4ae832f284163e424606bcefea89) (2026-09-01) | `dbt-core` [1.12.3](https://pypi.org/project/dbt-core/1.12.3/) |

> **Read the branch column carefully.** `dbt-labs/dbt-core`'s `main` branch is no
> longer the Python implementation — it now hosts **dbt Core v2.0 (beta), a
> ground-up rewrite in Rust** that underpins the Fusion engine. The Python dbt
> Core that ships as `dbt-core` on PyPI lives on the **`1.latest`** branch.
> Pinning `main` here would have documented a different engine from the one you
> are running.

### dbt-adapters: released 1.12.0 vs `main`

`main` reports `1.12.0rc1` in its `__version__.py`, while PyPI ships `1.12.0`.
Both were checked:

- **All incremental macro files are byte-identical** between released 1.12.0 and
  `main` at the pinned commit. Everything this reference says about generated SQL
  applies to both.
- **`_partition.py` differs in one place.** `main` adds a guard so that an
  `int64` range partition with `interval: 1` skips the `MOD` normalisation.
  Released 1.12.0 applies the normalisation unconditionally, which defeats
  partition elimination. Detailed in
  [balanced](docs/balanced/06-partition-config.md#int64-range-partitions-get-boundary-normalisation)
  and [expert](docs/expert/03-semantics.md#released-vs-main-delta).

### dbt-core: released 1.12.3 vs `1.latest`

The branch is at `1.14.0a1` while PyPI ships `1.12.3`, so the same check was run:

- **`microbatch.py` is byte-identical** between released 1.12.3 and `1.latest` at
  the pinned commit.
- The hook `transaction` default, the `lookback`/`begin`/`event_time` defaults,
  and the per-batch hook handling in `task/run.py` are identical in both.

Unless a page says otherwise, it describes **released `dbt-bigquery` 1.12.0 and
`dbt-core` 1.12.3** — what you get from `pip install`.

## Scope

BigQuery only. Other adapters implement the same strategy names with different
SQL, and the differences are not cosmetic — don't carry these conclusions across.

dbt Core v2.0 (the Rust rewrite) is **not** covered. Everything here describes
the Python implementation, which is what `pip install dbt-core` gives you today.

This is plain markdown with no build step. Read it on github.com, or clone it.

## Licence

Documentation, so [CC BY 4.0](LICENSE) — reuse it anywhere with attribution.
