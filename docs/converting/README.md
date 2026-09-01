# The conversion track

**Status: waves 1–3 complete — 63 of 138 units written.**

Five of the eleven parts are finished, covering the full path from *what have I
got* through to *the old script is retired*: assessment, write-pattern
archetypes, statement-level translation, hooks, and verification.

## Written so far

### Assess — *complete*

| | |
| --- | --- |
| [A1](A-assess/A1-inventory.md) | Inventory your scripts |
| [A2](A-assess/A2-map-dependencies.md) | Map declared dependencies against actual ones |
| [A3](A-assess/A3-classify-by-write-pattern.md) | Classify by write pattern |
| [A4](A-assess/A4-classify-by-trigger.md) | Classify by trigger |
| [A5](A-assess/A5-hidden-state.md) | Find the hidden state |
| [A6](A-assess/A6-compensating-hacks.md) | Find the compensating hacks |
| [A7](A-assess/A7-what-not-to-convert.md) | Decide what **not** to convert |
| [A8](A-assess/A8-estimate-risk.md) | Estimate conversion risk |
| [A9](A-assess/A9-correctness-baseline.md) | Capture the correctness baseline |

### Translate — *complete*

| | |
| --- | --- |
| [E1](E-translation/E1-one-statement-per-model.md) | **One statement per model** — start here |
| [E2](E-translation/E2-ordering-by-ref.md) | Ordering by `ref()` instead of line number |
| [E3](E-translation/E3-ref-vs-source.md) | `ref` vs `source`, and never both |
| [E4](E-translation/E4-cross-project-references.md) | Cross-project and cross-dataset references |
| [E5](E-translation/E5-finding-hardcoded-names.md) | Finding every hardcoded table name |
| [E6](E-translation/E6-hardcoded-dates.md) | Hardcoded dates and backfill parameters |
| [E7](E-translation/E7-idempotency-meaning.md) | Idempotency: what it means |
| [E8](E-translation/E8-idempotency-proving.md) | Idempotency: proving it |

### Write-pattern archetypes — *complete*

| | |
| --- | --- |
| [B1](B-write-patterns/B1-create-or-replace-to-table.md) | `CREATE OR REPLACE TABLE` → `table` |
| [B2](B-write-patterns/B2-create-if-not-exists.md) | `CREATE TABLE IF NOT EXISTS` bootstrap |
| [B3](B-write-patterns/B3-create-view.md) | `CREATE VIEW` → `view` |
| [B4](B-write-patterns/B4-materialized-views.md) | Materialized views |
| [B5](B-write-patterns/B5-unfiltered-insert.md) | Unfiltered `INSERT INTO ... SELECT` |
| [B6](B-write-patterns/B6-watermark-filter.md) | `INSERT` with a watermark filter |
| [B7](B-write-patterns/B7-external-watermark.md) | Watermark in a separate state table |
| [B8](B-write-patterns/B8-merge-on-clause-to-unique-key.md) | `MERGE`: `ON` clause → `unique_key` |
| [B9](B-write-patterns/B9-when-matched-update.md) | `MERGE`: `WHEN MATCHED THEN UPDATE` |
| [B10](B-write-patterns/B10-not-matched-by-source.md) | `MERGE`: `WHEN NOT MATCHED BY SOURCE THEN DELETE` |
| [B11](B-write-patterns/B11-conditional-when-matched.md) | `MERGE`: conditional `WHEN MATCHED AND ...` |
| [B12](B-write-patterns/B12-extra-predicates.md) | `MERGE`: extra `ON` predicates |
| [B13](B-write-patterns/B13-delete-insert-to-insert-overwrite.md) | `DELETE` + `INSERT` → `insert_overwrite` |
| [B14](B-write-patterns/B14-when-the-range-can-empty.md) | **When the range can legitimately empty** |
| [B15](B-write-patterns/B15-truncate-insert.md) | `TRUNCATE` + `INSERT` |
| [B16](B-write-patterns/B16-deduplication.md) | Deduplication scripts |

### Hooks — *complete*

| | |
| --- | --- |
| [F1](F-hooks/F1-what-a-hook-is.md) | What a hook actually is |
| [F2](F-hooks/F2-hook-rendering.md) | Hook rendering |
| [F3](F-hooks/F3-empty-hook-skipping.md) | Empty-hook skipping |
| [F4](F-hooks/F4-where-hooks-run.md) | Exactly where hooks run |
| [F5](F-hooks/F5-table-materialization-hooks.md) | Hooks in the `table` materialization |
| [F6](F-hooks/F6-transaction-filter.md) | The `transaction` filter |
| [F7](F-hooks/F7-hook-ordering.md) | Ordering within a hook list |
| [F8](F-hooks/F8-pre-hook-patterns.md) | pre-hook patterns worth keeping |
| [F9](F-hooks/F9-pre-hook-deletes.md) | pre-hook deletes — the common bad conversion |
| [F10](F-hooks/F10-post-hook-patterns.md) | post-hook patterns worth keeping |
| [F11](F-hooks/F11-grants-vs-post-hook.md) | post-hook vs the `grants` config |
| [F12](F-hooks/F12-post-hook-table-options.md) | post-hook: table options |
| [F13](F-hooks/F13-post-hook-audit-rows.md) | post-hook: audit rows |
| [F14](F-hooks/F14-on-run-start-end.md) | `on-run-start` / `on-run-end` |
| [F15](F-hooks/F15-hooks-and-temp-relation.md) | Hooks and the temp relation |
| [F16](F-hooks/F16-hooks-and-failure.md) | Hooks and failure semantics |
| [F17](F-hooks/F17-when-a-hook-is-wrong.md) | **When a hook is the wrong answer** |

### Proving correctness — *complete*

| | |
| --- | --- |
| [H1](H-verification/H1-what-correct-means.md) | What "correct" means here |
| [H2](H-verification/H2-row-count-parity.md) | Row-count parity |
| [H3](H-verification/H3-checksum-parity.md) | Checksum and hash parity |
| [H4](H-verification/H4-column-level-diffing.md) | Column-level diffing |
| [H5](H-verification/H5-shadow-mode.md) | Shadow mode |
| [H6](H-verification/H6-shadow-duration.md) | How long to shadow |
| [H7](H-verification/H7-reconciling-ordering.md) | Reconciling: ordering |
| [H8](H-verification/H8-reconciling-nulls.md) | Reconciling: nulls |
| [H9](H-verification/H9-reconciling-numeric-precision.md) | Reconciling: numeric precision |
| [H10](H-verification/H10-reconciling-timestamps.md) | Reconciling: timestamps |
| [H11](H-verification/H11-differences-that-should-exist.md) | Differences that should exist |
| [H12](H-verification/H12-tests-from-guarantees.md) | Tests from the script's guarantees |
| [H13](H-verification/H13-sign-off.md) | Sign-off and retiring the script |

## Where to start

Read [E1](E-translation/E1-one-statement-per-model.md), then
[A3](A-assess/A3-classify-by-write-pattern.md) to find your archetype, then the
matching Part B page.

If your script is a `DELETE`+`INSERT`, **[B14](B-write-patterns/B14-when-the-range-can-empty.md)
is not optional** — the obvious conversion is silently wrong.

The rest of the map follows.

Turning existing scripts — scheduled queries, stored procedures, `bq` shell
pipelines, Python jobs — into dbt models. The existing three tracks describe how
incremental models *behave*. This track is about how you *get there* from what
you already have running in production.

It is scoped deliberately wider than incremental models, because a real
conversion is never only about the write pattern. It's about ordering,
idempotency, hooks, parameters, backfills, and proving the new thing matches the
old thing before you delete anything.

## Why it's a separate track

The other tracks answer "what does dbt do?". This one answers "what do I do with
this 400-line stored procedure?" — a different question with a different failure
mode. Getting incremental strategy right and still producing a broken conversion
is entirely possible, usually by faithfully porting a bug or by rebuilding the
script's imperative structure in Jinja.

## Scope

**In scope**

- Every common script archetype and the dbt shape it maps to
- Statement-level translation: temp tables, variables, dynamic SQL, control flow
- Hooks in depth — pre, post, `on-run-start`/`on-run-end` — including where they
  actually run and when they're the wrong tool
- Scheduling, parameters, and backfills after conversion
- **Proving the conversion is correct**, which is the part everyone skips
- Migration sequencing and decommissioning
- Anti-patterns

**Out of scope**

- Non-BigQuery warehouses, as everywhere in this repo
- dbt project setup, profiles, and CI — well covered by the official docs
- Orchestration tooling comparisons

## How this track is sourced

This matters, because the epistemics differ from the rest of the repo.

The other three tracks are source-derived: every claim traces to the adapter code
at a pinned commit. A conversion guide cannot work that way — most of it is craft.
So every unit in the backlog carries a tag:

| Tag | Meaning |
| --- | --- |
| `SRC` | Verifiable against `dbt-labs/dbt-adapters` at the [pinned commit](../../README.md). Held to the same standard as the rest of the repo. |
| `CORE` | Depends on dbt-core, now pinned at `1.latest` @ `300e80c` / released 1.12.3. Verifiable; the specific behaviour still needs reading. |
| `CORE✓` | dbt-core behaviour already verified. Ready to write. |
| `CRAFT` | Judgment and practice. Not derivable from any source. Will be written as advice, and labelled as such. |

Mixing these silently would undermine the thing that makes this repo worth
reading. Pages will say which they are.

## The eleven parts

| Part | Subject | Units |
| --- | --- | --- |
| **A** | [Assess before you convert](BACKLOG.md#part-a--assess-before-you-convert) | 9 |
| **B** | [Write-pattern archetypes](BACKLOG.md#part-b--write-pattern-archetypes) | 16 |
| **C** | [Structural archetypes](BACKLOG.md#part-c--structural-archetypes) | 14 |
| **D** | [Data movement, DDL, metadata](BACKLOG.md#part-d--data-movement-ddl-and-metadata) | 14 |
| **E** | [Statement-level translation](BACKLOG.md#part-e--statement-level-translation) | 13 |
| **F** | [Hooks](BACKLOG.md#part-f--hooks) | 17 |
| **G** | [Scheduling, parameters, backfills](BACKLOG.md#part-g--scheduling-parameters-backfills) | 11 |
| **H** | [Proving correctness](BACKLOG.md#part-h--proving-correctness) | 13 |
| **I** | [Migration strategy](BACKLOG.md#part-i--migration-strategy) | 10 |
| **J** | [Operating it afterwards](BACKLOG.md#part-j--operating-it-afterwards) | 9 |
| **K** | [Anti-patterns](BACKLOG.md#part-k--anti-patterns) | 12 |

**138 units total**, none larger than a single page — if a unit would need a long
page, it is already split. Full breakdown with sizes, sourcing and dependencies
in [the backlog](BACKLOG.md), which also sets out
[five delivery waves](BACKLOG.md#delivery-waves).

## Findings already banked

Verified in source while scoping this track, so Part D starts from fact rather
than folklore:

- **Every BigQuery materialization calls `run_hooks` exactly once per phase**, at
  the default `inside_transaction=True`. The default non-BigQuery materialization
  calls it four times. Consequence: a hook configured `transaction: false` is
  filtered out by `selectattr` and **silently never runs** on BigQuery.
- **Post-hooks run before `apply_grants`, `persist_docs`, and the temp-relation
  drop.** So a post-hook can still see the temp table, and must not assume grants
  have been applied.
- **Hooks that render to empty or whitespace are skipped entirely**
  (`if (rendered | length) > 0`), which is why a conditional hook that produces
  nothing is a no-op rather than an error.

Since dbt-core was pinned, three more, from `dbt-core` at `1.latest` @ `300e80c`:

- **`Hook.transaction` defaults to `True`**, which is what every BigQuery
  materialization passes. So ordinary hooks are fine; the trap above is narrow,
  hitting only an explicit `transaction: false`.
- **Microbatch hooks fire once per model, not once per batch** — `pre_hook` on
  the first batch, `post_hook` on the last. A 400-batch backfill runs each once.
- **A microbatch checkpoint sitting exactly on a batch boundary silently
  increments `lookback` by one**, so that run reprocesses one batch more than
  configured.

## Where to start

The backlog opens with a [ten-unit first wave](BACKLOG.md#wave-1--foundation-10-units)
that leaves the track coherent and useful before the other 128 exist. It covers
the two conversions people actually arrive with — a hand-written `MERGE` and a
`DELETE`+`INSERT` — end to end, with the reasoning and the verification either
side of them.

**That blocker is cleared.** dbt-core is now pinned alongside dbt-adapters, and
four of the seventeen dependent units are already verified. Thirteen still need
reading, but none are blocked. See
[the backlog](BACKLOG.md#dbt-core-is-now-pinned).

---

Back to [the repository README](../../README.md)
