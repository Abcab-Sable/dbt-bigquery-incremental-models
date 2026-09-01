# The conversion track

**Status: complete — all 138 units across eleven parts.**

The full arc: what you have, whether to convert it, how every construct maps,
how to prove the result, how to cut over, how to operate it afterwards, and what
not to do.

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
| [E9](E-translation/E9-session-settings.md) | Session settings |
| [E10](E-translation/E10-system-variables.md) | System variables |
| [E11](E-translation/E11-query-parameters.md) | Query parameters → vars |
| [E12](E-translation/E12-cost-controls.md) | Cost controls |
| [E13](E-translation/E13-time-travel.md) | Time travel |

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

### Structural archetypes — *complete*

| | |
| --- | --- |
| [C1](C-structural/C1-multi-statement-to-ctes.md) | Multi-statement → CTEs |
| [C2](C-structural/C2-ephemeral-models.md) | Multi-statement → ephemeral models |
| [C3](C-structural/C3-separate-models.md) | Multi-statement → separate models |
| [C4](C-structural/C4-fan-out.md) | Scripts writing to several tables |
| [C5](C-structural/C5-declare-set-variables.md) | `DECLARE` / `SET` variables |
| [C6](C-structural/C6-if-branching.md) | `IF` / `ELSEIF` branching |
| [C7](C-structural/C7-loops.md) | `WHILE` / `LOOP` iteration |
| [C8](C-structural/C8-exception-handling.md) | `EXCEPTION WHEN ERROR` |
| [C9](C-structural/C9-transactions.md) | `BEGIN TRANSACTION` / `COMMIT` |
| [C10](C-structural/C10-dynamic-sql.md) | `EXECUTE IMMEDIATE` and dynamic SQL |
| [C11](C-structural/C11-temp-functions.md) | `CREATE TEMP FUNCTION` |
| [C12](C-structural/C12-nested-procedures.md) | Procedures calling procedures |
| [C13](C-structural/C13-python-scripts.md) | Python scripts |
| [C14](C-structural/C14-orchestration.md) | Shell, `bq` CLI, Airflow |

### Data movement, DDL, metadata — *complete*

| | |
| --- | --- |
| [D1](D-data-movement/D1-load-data.md) | `LOAD DATA` from GCS |
| [D2](D-data-movement/D2-export-data.md) | `EXPORT DATA` to GCS |
| [D3](D-data-movement/D3-external-tables.md) | External tables and BigLake |
| [D4](D-data-movement/D4-wildcard-tables.md) | Wildcard tables and `_TABLE_SUFFIX` |
| [D5](D-data-movement/D5-sharded-tables.md) | Date-sharded tables → partitioned |
| [D6](D-data-movement/D6-partitioning-ddl.md) | Partitioning and clustering DDL |
| [D7](D-data-movement/D7-table-options.md) | Expiration, labels, description |
| [D8](D-data-movement/D8-add-column-migrations.md) | `ALTER TABLE ADD COLUMN` |
| [D9](D-data-movement/D9-column-type-changes.md) | Column type changes |
| [D10](D-data-movement/D10-grants-authorized-views.md) | Grants and authorized views |
| [D11](D-data-movement/D11-policy-tags-rls.md) | Policy tags and row-level security |
| [D12](D-data-movement/D12-assert-gates.md) | `ASSERT` gates → dbt tests |
| [D13](D-data-movement/D13-notifications.md) | Notification side-effects |
| [D14](D-data-movement/D14-audit-writes.md) | Audit and metadata writes |

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

### Scheduling, parameters, backfills — *complete*

| | |
| --- | --- |
| [G1](G-scheduling/G1-cron-to-dbt-build.md) | From cron to `dbt build` |
| [G2](G-scheduling/G2-consolidating-schedules.md) | Consolidating schedules |
| [G3](G-scheduling/G3-passing-dates.md) | Passing dates in |
| [G4](G-scheduling/G4-env-vars-secrets.md) | Environment variables and secrets |
| [G5](G-scheduling/G5-backfill-full-refresh.md) | Backfill via `--full-refresh` |
| [G6](G-scheduling/G6-backfill-microbatch.md) | Backfill via microbatch |
| [G7](G-scheduling/G7-backfill-partition-ranges.md) | Backfill via partition ranges |
| [G8](G-scheduling/G8-late-arriving-data.md) | Late-arriving data |
| [G9](G-scheduling/G9-selectors.md) | Selectors |
| [G10](G-scheduling/G10-state-selection.md) | State-based selection and slim CI |
| [G11](G-scheduling/G11-retry-and-failure.md) | Retry and partial-failure semantics |

### Migration strategy — *complete*

| | |
| --- | --- |
| [I1](I-migration/I1-conversion-order.md) | Conversion order |
| [I2](I-migration/I2-strangler-pattern.md) | The strangler pattern |
| [I3](I-migration/I3-converting-with-dependents.md) | Converting with dependents |
| [I4](I-migration/I4-dual-write.md) | Dual-write during cutover |
| [I5](I-migration/I5-notifying-consumers.md) | Telling downstream consumers |
| [I6](I-migration/I6-rollback-keeping-script.md) | Rollback: keeping the script |
| [I7](I-migration/I7-rollback-non-viable.md) | When rollback stops being viable |
| [I8](I-migration/I8-decommissioning.md) | Decommissioning checklist |
| [I9](I-migration/I9-what-to-keep.md) | What to keep from the old script |
| [I10](I-migration/I10-documenting-decisions.md) | Documenting the decisions |

### Operating it afterwards — *complete*

| | |
| --- | --- |
| [J1](J-operating/J1-cost-after-conversion.md) | Cost after conversion |
| [J2](J-operating/J2-monitoring-drift.md) | Monitoring incremental drift |
| [J3](J-operating/J3-scheduled-reconciliation.md) | **Scheduled reconciliation** — the control that works |
| [J4](J-operating/J4-alerting.md) | Alerting |
| [J5](J-operating/J5-ownership-handover.md) | Ownership and handover |
| [J6](J-operating/J6-freshness-checks.md) | Freshness checks |
| [J7](J-operating/J7-schema-evolution.md) | Schema evolution |
| [J8](J-operating/J8-partition-growth.md) | Partition-count growth |
| [J9](J-operating/J9-revisiting-strategy.md) | Revisiting the strategy choice |

### Anti-patterns — *complete*

| | |
| --- | --- |
| [K1](K-antipatterns/K1-mega-model.md) | The mega-model |
| [K2](K-antipatterns/K2-hooks-as-escape-hatch.md) | Hooks as an escape hatch |
| [K3](K-antipatterns/K3-unnecessary-incremental.md) | Incremental where a table belongs |
| [K4](K-antipatterns/K4-run-operation-as-scheduler.md) | `run-operation` as a scheduler |
| [K5](K-antipatterns/K5-imperative-jinja.md) | Imperative structure in Jinja |
| [K6](K-antipatterns/K6-porting-the-bug.md) | Porting the bug faithfully |
| [K7](K-antipatterns/K7-over-parameterising.md) | Over-parameterising with vars |
| [K8](K-antipatterns/K8-one-model-per-statement.md) | One model per statement |
| [K9](K-antipatterns/K9-ephemeral-overuse.md) | Ephemeral overuse |
| [K10](K-antipatterns/K10-no-tests.md) | No tests, because the script had none |
| [K11](K-antipatterns/K11-convert-and-optimise.md) | Converting and optimising together |
| [K12](K-antipatterns/K12-trusting-green-runs.md) | **Trusting a green run** |

## Where to start

**In a hurry, or converting a specific statement?** The
[statement-pattern cheatsheet](CHEATSHEET.md) is the whole track compressed into
one page: statement shape → strategy → config, `MERGE` clause by clause, and the
edge cases ranked by how often they cost a day. Every row links back here.

Otherwise: read [E1](E-translation/E1-one-statement-per-model.md), then
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
| **A** | [Assess before you convert](#assess--complete) | 9 |
| **B** | [Write-pattern archetypes](#translate--complete) | 16 |
| **C** | [Structural archetypes](#write-pattern-archetypes--complete) | 14 |
| **D** | [Data movement, DDL, metadata](#structural-archetypes--complete) | 14 |
| **E** | [Statement-level translation](#data-movement-ddl-metadata--complete) | 13 |
| **F** | [Hooks](#hooks--complete) | 17 |
| **G** | [Scheduling, parameters, backfills](#proving-correctness--complete) | 11 |
| **H** | [Proving correctness](#scheduling-parameters-backfills--complete) | 13 |
| **I** | [Migration strategy](#migration-strategy--complete) | 10 |
| **J** | [Operating it afterwards](#operating-it-afterwards--complete) | 9 |
| **K** | [Anti-patterns](#anti-patterns--complete) | 12 |

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
