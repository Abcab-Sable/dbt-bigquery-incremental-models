# Conversion track — backlog

Decomposition of [the conversion track](README.md) into **138 units across
eleven parts**.

## Rules this backlog follows

**No unit is large.** If a unit would need a long page, it is already split.
Sizes are `S` (a section) or `M` (a page). There is no `L` — that's the
granularity rule, not an accident.

**Every unit answers one question.** If it answers two, it's two units.

**Sourcing is declared, never implied:**

| Tag | Meaning |
| --- | --- |
| `SRC` | Verifiable against dbt-adapters at the [pinned commit](../../README.md). Same standard as the rest of the repo. |
| `CORE` | Depends on dbt-core, now [pinned](../../README.md) at `1.latest` @ `300e80c` / released 1.12.3. Verifiable, but the specific behaviour still needs reading. |
| `CORE✓` | dbt-core behaviour **already verified** in source. Ready to write. |
| `CRAFT` | Judgment and practice. Will be written as advice and labelled as such. |

**Status** — ✅ means written and linked. Everything else is `todo`.
**96 of 138 done** — waves 1–4 complete. Parts A, B, C, D, E, F and H are finished.

**Deps** — units that should be written first, because this one links to them.

Jump to: [A](#part-a--assess-before-you-convert) ·
[B](#part-b--write-pattern-archetypes) · [C](#part-c--structural-archetypes) ·
[D](#part-d--data-movement-ddl-and-metadata) ·
[E](#part-e--statement-level-translation) · [F](#part-f--hooks) ·
[G](#part-g--scheduling-parameters-backfills) ·
[H](#part-h--proving-correctness) · [I](#part-i--migration-strategy) ·
[J](#part-j--operating-it-afterwards) · [K](#part-k--anti-patterns) ·
[Waves](#delivery-waves)

---

## Part A — Assess before you convert

Nothing here produces a model. It stops you converting the wrong thing, or
converting the right thing without knowing what "correct" meant.

| ID | Unit | Size | Sourcing | Deps |
| --- | --- | --- | --- | --- |
| **A1** ✅ | [Inventory: enumerate every script, its schedule, and its owner](A-assess/A1-inventory.md) | S | CRAFT | — |
| **A2** ✅ | [Map declared dependencies against actual ones](A-assess/A2-map-dependencies.md) | M | CRAFT | A1 |
| **A3** ✅ | [Classify by write pattern: replace / append / upsert / delete-insert / DDL / side-effect](A-assess/A3-classify-by-write-pattern.md) | M | CRAFT | — |
| **A4** ✅ | [Classify by trigger: cron, event-driven, manual, chained](A-assess/A4-classify-by-trigger.md) | S | CRAFT | A1 |
| **A5** ✅ | [Find the hidden state: manual steps, assumed-existing objects](A-assess/A5-hidden-state.md) | M | CRAFT | — |
| **A6** ✅ | [Find compensating hacks — the fixes that encode an upstream bug](A-assess/A6-compensating-hacks.md) | M | CRAFT | A5 |
| **A7** ✅ | [Decide what **not** to convert](A-assess/A7-what-not-to-convert.md) | S | CRAFT | A3 |
| **A8** ✅ | [Estimate conversion risk per script](A-assess/A8-estimate-risk.md) | S | CRAFT | A3, A5 |
| **A9** ✅ | [Capture the correctness baseline before touching anything](A-assess/A9-correctness-baseline.md) | M | CRAFT | — |

A3 is the spine of Parts B–D: classification picks the archetype page.
A6 and A9 are what make Part H possible — without them you cannot later tell a
bug from a feature.

---

## Part B — Write-pattern archetypes

The DML conversions. Each unit: the script pattern, the dbt shape, a worked
before/after, and what breaks in translation.

| ID | Unit | Size | Sourcing | Deps |
| --- | --- | --- | --- | --- |
| **B1** ✅ | [`CREATE OR REPLACE TABLE ... AS SELECT` → `materialized='table'`](B-write-patterns/B1-create-or-replace-to-table.md) | S | SRC | E1 |
| **B2** ✅ | [`CREATE TABLE IF NOT EXISTS` bootstrap → letting dbt own creation](B-write-patterns/B2-create-if-not-exists.md) | S | SRC | B1 |
| **B3** ✅ | [`CREATE [OR REPLACE] VIEW` → `materialized='view'`](B-write-patterns/B3-create-view.md) | S | SRC | — |
| **B4** ✅ | [Materialized view managed by script → the MV materialization](B-write-patterns/B4-materialized-views.md) | M | SRC | B3 |
| **B5** ✅ | [Unfiltered `INSERT INTO ... SELECT` → incremental, append semantics](B-write-patterns/B5-unfiltered-insert.md) | M | SRC | E1 |
| **B6** ✅ | [`INSERT` with a watermark filter → incremental + `is_incremental()`](B-write-patterns/B6-watermark-filter.md) | M | SRC | B5 |
| **B7** ✅ | [Watermark held in a separate state table → replacing it with `{{ this }}`](B-write-patterns/B7-external-watermark.md) | M | CRAFT | B6 |
| **B8** ✅ | [Hand-written `MERGE`: the `ON` clause → `unique_key`](B-write-patterns/B8-merge-on-clause-to-unique-key.md) | M | SRC | — |
| **B9** ✅ | [Hand-written `MERGE`: `WHEN MATCHED THEN UPDATE` → `merge_update_columns` / `merge_exclude_columns`](B-write-patterns/B9-when-matched-update.md) | M | SRC | B8 |
| **B10** ✅ | [Hand-written `MERGE`: `WHEN NOT MATCHED BY SOURCE THEN DELETE` → the case dbt's `merge` cannot express](B-write-patterns/B10-not-matched-by-source.md) | M | SRC | B8 |
| **B11** ✅ | [Hand-written `MERGE`: conditional `WHEN MATCHED AND ...` clauses](B-write-patterns/B11-conditional-when-matched.md) | M | SRC | B9 |
| **B12** ✅ | [Hand-written `MERGE`: extra `ON` predicates → `incremental_predicates`](B-write-patterns/B12-extra-predicates.md) | M | SRC | B8 |
| **B13** ✅ | [`DELETE` + `INSERT` over a date range → `insert_overwrite`](B-write-patterns/B13-delete-insert-to-insert-overwrite.md) | M | SRC | — |
| **B14** ✅ | [`DELETE` + `INSERT` where the range can legitimately empty → **the behaviour change**](B-write-patterns/B14-when-the-range-can-empty.md) | M | SRC | B13 |
| **B15** ✅ | [`TRUNCATE` + `INSERT` → `table`, or `insert_overwrite` when partitioned](B-write-patterns/B15-truncate-insert.md) | S | SRC | B13 |
| **B16** ✅ | [Deduplication scripts (`QUALIFY ROW_NUMBER()`) → where dedup belongs after conversion](B-write-patterns/B16-deduplication.md) | M | CRAFT | B8 |

**B14 is the most important unit in the track.** A `DELETE`+`INSERT` script
genuinely empties its range; the naive `insert_overwrite` conversion does not,
because of [the empty-partition trap](../balanced/04-insert-overwrite.md#the-empty-partition-trap).
That is a silent behaviour change *introduced by the conversion itself*.

**B10 is the honest-answer unit.** dbt's `merge` strategy emits no
`not matched by source` clause at all — a script relying on it is asking for
`insert_overwrite` semantics, or for a redesign. Say so plainly.

---

## Part C — Structural archetypes

Scripts whose difficulty is shape, not write pattern.

| ID | Unit | Size | Sourcing | Deps |
| --- | --- | --- | --- | --- |
| **C1** ✅ | [Multi-statement script → CTEs in one model](C-structural/C1-multi-statement-to-ctes.md) | M | CRAFT | E1 |
| **C2** ✅ | [Multi-statement script → ephemeral models](C-structural/C2-ephemeral-models.md) | M | CORE | C1 |
| **C3** ✅ | [Multi-statement script → separate materialized models, and when that's right](C-structural/C3-separate-models.md) | M | CRAFT | C1 |
| **C4** ✅ | [One script writing to several tables → fan-out into models](C-structural/C4-fan-out.md) | M | CRAFT | C3 |
| **C5** ✅ | [`DECLARE` / `SET` scalar variables → vars, macros, `_dbt_max_partition`](C-structural/C5-declare-set-variables.md) | M | SRC | E9 |
| **C6** ✅ | [`IF` / `ELSEIF` branching → Jinja conditionals, or a redesign](C-structural/C6-if-branching.md) | M | CRAFT | C5 |
| **C7** ✅ | [`WHILE` / `LOOP` iteration → why it mostly can't map](C-structural/C7-loops.md) | M | CRAFT | C6 |
| **C8** ✅ | [`BEGIN ... EXCEPTION WHEN ERROR` → dbt's failure model instead](C-structural/C8-exception-handling.md) | M | CRAFT | G11 |
| **C9** ✅ | [`BEGIN TRANSACTION` / `COMMIT` → what BigQuery and dbt actually guarantee](C-structural/C9-transactions.md) | M | SRC | F6 |
| **C10** ✅ | [`EXECUTE IMMEDIATE` and dynamic SQL → macros, or a deliberate `run-operation`](C-structural/C10-dynamic-sql.md) | M | CRAFT | — |
| **C11** ✅ | [`CREATE TEMP FUNCTION` → macros vs persistent UDFs](C-structural/C11-temp-functions.md) | M | CRAFT | — |
| **C12** ✅ | [Stored procedures calling other procedures → the DAG they were imitating](C-structural/C12-nested-procedures.md) | M | CRAFT | E2 |
| **C13** ✅ | [Python script using the BigQuery client → Python model, seed, or stays external](C-structural/C13-python-scripts.md) | M | CORE | — |
| **C14** ✅ | [Shell / `bq` CLI / Airflow orchestration → the `ref()` DAG](C-structural/C14-orchestration.md) | M | CRAFT | E2 |

C7 needs an honest answer rather than a clever one: most procedural loops are a
set-based operation in disguise, or genuine orchestration that belongs outside
dbt. Document both exits.

---

## Part D — Data movement, DDL, and metadata

Everything a script does that isn't computing rows.

| ID | Unit | Size | Sourcing | Deps |
| --- | --- | --- | --- | --- |
| **D1** ✅ | [`LOAD DATA` from GCS → seeds, external tables, or stays external](D-data-movement/D1-load-data.md) | M | CORE | — |
| **D2** ✅ | [`EXPORT DATA` to GCS → post-hook vs external orchestration](D-data-movement/D2-export-data.md) | M | CRAFT | F10 |
| **D3** ✅ | [External tables and BigLake → sources, not models](D-data-movement/D3-external-tables.md) | M | CRAFT | E3 |
| **D4** ✅ | [Wildcard tables and `_TABLE_SUFFIX`](D-data-movement/D4-wildcard-tables.md) | M | CRAFT | D5 |
| **D5** ✅ | [Date-sharded tables (`events_20260831`) → one partitioned table](D-data-movement/D5-sharded-tables.md) | M | CRAFT | — |
| **D6** ✅ | [Partitioning and clustering DDL → `partition_by` / `cluster_by` config](D-data-movement/D6-partitioning-ddl.md) | S | SRC | — |
| **D7** ✅ | [Expiration, labels, description → config vs post-hook](D-data-movement/D7-table-options.md) | M | CRAFT | F12 |
| **D8** ✅ | [`ALTER TABLE ADD COLUMN` migrations → `on_schema_change`](D-data-movement/D8-add-column-migrations.md) | M | SRC | — |
| **D9** ✅ | [Column type changes → `on_schema_change` and its limits](D-data-movement/D9-column-type-changes.md) | M | SRC | D8 |
| **D10** ✅ | [Grants and authorized views → the `grants` config](D-data-movement/D10-grants-authorized-views.md) | M | SRC | F11 |
| **D11** ✅ | [Policy tags and row-level security](D-data-movement/D11-policy-tags-rls.md) | S | CRAFT | D10 |
| **D12** ✅ | [`ASSERT` data-quality gates → dbt tests](D-data-movement/D12-assert-gates.md) | M | CRAFT | H12 |
| **D13** ✅ | [Notification side-effects (Pub/Sub, email) → out of dbt, usually](D-data-movement/D13-notifications.md) | S | CRAFT | F17 |
| **D14** ✅ | [Audit and metadata table writes](D-data-movement/D14-audit-writes.md) | M | CRAFT | F13 |

D5 is a bigger win than it looks. Sharded-table scripts are extremely common and
converting them to a partitioned table changes the cost model entirely — worth
its own before/after.

---

## Part E — Statement-level translation

The mechanical problems recurring inside every conversion, factored out so B–D
can link instead of repeating.

| ID | Unit | Size | Sourcing | Deps |
| --- | --- | --- | --- | --- |
| **E1** ✅ | [**One statement per model** — the constraint everything follows from](E-translation/E1-one-statement-per-model.md) | S | SRC | — |
| **E2** ✅ | [Ordering by `ref()` instead of by line number](E-translation/E2-ordering-by-ref.md) | M | CORE | E1 |
| **E3** ✅ | [`ref` vs `source`, and never both for the same object](E-translation/E3-ref-vs-source.md) | S | CORE | E2 |
| **E4** ✅ | [Cross-project and cross-dataset references](E-translation/E4-cross-project-references.md) | S | CRAFT | E3 |
| **E5** ✅ | [Finding every hardcoded table name](E-translation/E5-finding-hardcoded-names.md) | S | CRAFT | E3 |
| **E6** ✅ | [Hardcoded dates and manual backfill parameters](E-translation/E6-hardcoded-dates.md) | S | CRAFT | G3 |
| **E7** ✅ | [Idempotency: what it means for a converted model](E-translation/E7-idempotency-meaning.md) | M | CRAFT | — |
| **E8** ✅ | [Idempotency: proving it](E-translation/E8-idempotency-proving.md) | M | CRAFT | E7, H2 |
| **E9** ✅ | [Session settings that spanned statements](E-translation/E9-session-settings.md) | M | SRC | — |
| **E10** ✅ | [System variables (`@@dataset_id`, `@@project_id`)](E-translation/E10-system-variables.md) | S | CRAFT | E9 |
| **E11** ✅ | [Query parameters (`@param`) → vars](E-translation/E11-query-parameters.md) | S | CORE | G3 |
| **E12** ✅ | [Cost controls (`maximum_bytes_billed`) after conversion](E-translation/E12-cost-controls.md) | S | CRAFT | J1 |
| **E13** ✅ | [Time travel and `FOR SYSTEM_TIME AS OF`](E-translation/E13-time-travel.md) | S | CRAFT | — |

E1 is the conceptual hinge of the entire track. A script is a sequence of
statements; a model is one `select`. Nearly every "how do I do X in dbt" question
during a conversion is downstream of that one constraint.

---

## Part F — Hooks

Deep, because hooks are where conversions go to die: the script's leftover
statements get swept into a `post_hook` and the model quietly becomes a script
again.

Three units start from already-verified fact — see
[the track index](README.md#findings-already-banked).

| ID | Unit | Size | Sourcing | Deps |
| --- | --- | --- | --- | --- |
| **F1** ✅ | [What a hook is: `run_hooks` mechanics and the `selectattr` filter](F-hooks/F1-what-a-hook-is.md) | M | SRC | — |
| **F2** ✅ | [Hook rendering: when the Jinja inside a hook is evaluated](F-hooks/F2-hook-rendering.md) | M | SRC | F1 |
| **F3** ✅ | [Empty-hook skipping — why a conditional hook that renders blank is a no-op](F-hooks/F3-empty-hook-skipping.md) | S | SRC | F2 |
| **F4** ✅ | [Exactly where hooks run in the BigQuery **incremental** materialization](F-hooks/F4-where-hooks-run.md) | M | SRC | F1 |
| **F5** ✅ | [Where hooks run in the BigQuery **table** materialization, for contrast](F-hooks/F5-table-materialization-hooks.md) | S | SRC | F4 |
| **F6** ✅ | [The `transaction` filter, and why `transaction: false` never fires on BigQuery](F-hooks/F6-transaction-filter.md) | M | SRC + CORE✓ | F1 |
| **F7** ✅ | [Ordering within a hook list](F-hooks/F7-hook-ordering.md) | S | SRC | F1 |
| **F8** ✅ | [pre-hook: the patterns worth keeping](F-hooks/F8-pre-hook-patterns.md) | M | CRAFT | F4 |
| **F9** ✅ | [pre-hook: deleting rows before insert — and why it's usually the wrong shape](F-hooks/F9-pre-hook-deletes.md) | M | CRAFT | F8, B13 |
| **F10** ✅ | [post-hook: the patterns worth keeping](F-hooks/F10-post-hook-patterns.md) | M | CRAFT | F4 |
| **F11** ✅ | [post-hook vs the `grants` config, and the ordering trap](F-hooks/F11-grants-vs-post-hook.md) | M | SRC | F10 |
| **F12** ✅ | [post-hook: table options and metadata](F-hooks/F12-post-hook-table-options.md) | S | CRAFT | F10 |
| **F13** ✅ | [post-hook: writing audit rows](F-hooks/F13-post-hook-audit-rows.md) | M | CRAFT | F10 |
| **F14** ✅ | [`on-run-start` / `on-run-end` vs per-model hooks](F-hooks/F14-on-run-start-end.md) | M | CORE✓ | F10 |
| **F15** ✅ | [Hooks that reference the temp relation — what's still alive when](F-hooks/F15-hooks-and-temp-relation.md) | M | SRC | F4 |
| **F16** ✅ | [Hooks and failure semantics: what runs when the model fails](F-hooks/F16-hooks-and-failure.md) | M | CORE✓ | F4 |
| **F17** ✅ | [**When a hook is the wrong answer**](F-hooks/F17-when-a-hook-is-wrong.md) | M | CRAFT | F8, F10 |

**F6 is now fully resolved, and the answer narrows the trap.** dbt-core defines
`Hook.transaction: bool = True` (`artifacts/resources/v1/config.py`), so a
plain-string hook defaults to `True` — which matches the only `inside_transaction`
value any BigQuery materialization passes. **Ordinary hooks are unaffected.** Only
an explicit `transaction: false` is filtered out by `selectattr` and silently
never runs. Write it as a narrow, real trap rather than a broad one.

**F16 is partly resolved.** For microbatch models, `task/run.py` clears
`pre_hook` on every batch except the first and `post_hook` on every batch except
the last — hooks fire once per model, not once per batch. What still needs
reading is what happens to a post-hook when the model itself fails.

**F11's trap is source-verified:** post-hooks run *before* `apply_grants`, so a
post-hook granting access races the config rather than composing with it.

**F17 should be written early.** Everything else in Part F explains how hooks
work; this one explains why the answer is usually a model.

---

## Part G — Scheduling, parameters, backfills

| ID | Unit | Size | Sourcing | Deps |
| --- | --- | --- | --- | --- |
| G1 | From cron entries to `dbt build` | M | CRAFT | — |
| G2 | Consolidating many schedules into one run | M | CRAFT | G1, E2 |
| G3 | Passing dates in: `vars` and `--vars` | M | CORE | — |
| G4 | Environment variables and secrets | S | CORE | G3 |
| G5 | Backfill via `--full-refresh` | M | SRC | — |
| G6 | Backfill via microbatch | M | CORE✓ | G5 |
| G7 | Backfill via explicit partition ranges | M | SRC | G5, B13 |
| G8 | Late-arriving data after conversion | M | CRAFT | B6 |
| G9 | Selectors: `--select`, `--exclude` | S | CORE | — |
| G10 | State-based selection and slim CI | M | CORE | G9 |
| G11 | Retry and partial-failure semantics vs a half-completed script | M | CRAFT | — |

G11 is underrated. A failed script often leaves partial writes; a failed dbt run
has different guarantees per materialization. Conversion changes your failure
modes and nobody documents that.

---

## Part H — Proving correctness

The part everyone skips. A conversion isn't done when the model runs; it's done
when you can show it matches.

| ID | Unit | Size | Sourcing | Deps |
| --- | --- | --- | --- | --- |
| **H1** ✅ | [What "correct" means for this particular conversion](H-verification/H1-what-correct-means.md) | M | CRAFT | A9 |
| **H2** ✅ | [Row-count parity](H-verification/H2-row-count-parity.md) | S | CRAFT | H1 |
| **H3** ✅ | [Checksum and hash parity](H-verification/H3-checksum-parity.md) | M | CRAFT | H2 |
| **H4** ✅ | [Column-level diffing](H-verification/H4-column-level-diffing.md) | M | CRAFT | H3 |
| **H5** ✅ | [Shadow mode: running old and new in parallel](H-verification/H5-shadow-mode.md) | M | CRAFT | H1 |
| **H6** ✅ | [Shadow mode: how long, and what ends it](H-verification/H6-shadow-duration.md) | S | CRAFT | H5 |
| **H7** ✅ | [Reconciling: row and column ordering](H-verification/H7-reconciling-ordering.md) | S | CRAFT | H4 |
| **H8** ✅ | [Reconciling: null vs empty string vs absent](H-verification/H8-reconciling-nulls.md) | S | CRAFT | H4 |
| **H9** ✅ | [Reconciling: float and `NUMERIC` precision](H-verification/H9-reconciling-numeric-precision.md) | S | CRAFT | H4 |
| **H10** ✅ | [Reconciling: timestamp precision and timezones](H-verification/H10-reconciling-timestamps.md) | S | CRAFT | H4 |
| **H11** ✅ | [Reconciling: differences that are *supposed* to exist](H-verification/H11-differences-that-should-exist.md) | M | CRAFT | H4, B14 |
| **H12** ✅ | [Encoding the script's implicit guarantees as dbt tests](H-verification/H12-tests-from-guarantees.md) | M | CORE | H1 |
| **H13** ✅ | [Sign-off criteria, and when it's safe to delete the old script](H-verification/H13-sign-off.md) | S | CRAFT | H6, I8 |

H7–H10 exist separately because this is where conversions stall. Differences are
usually legitimate, and without a triage framework people either chase noise for
a week or wave through a real bug.

**H11 is the counterpart to B14** — sometimes the new output is *correctly*
different, and you need to be able to say why.

---

## Part I — Migration strategy

| ID | Unit | Size | Sourcing | Deps |
| --- | --- | --- | --- | --- |
| I1 | Conversion order: leaves first or roots first | M | CRAFT | A2 |
| I2 | The strangler pattern for a script suite | M | CRAFT | I1 |
| I3 | Converting one script while others still depend on its output | M | CRAFT | I2 |
| I4 | Dual-write during cutover | M | CRAFT | H5 |
| I5 | Telling downstream consumers | S | CRAFT | I4 |
| I6 | Rollback: keeping the old script runnable | S | CRAFT | I4 |
| I7 | Rollback: what makes it non-viable, and when you pass that point | M | CRAFT | I6 |
| I8 | Decommissioning checklist | S | CRAFT | I7 |
| I9 | What to keep from the old script: comments, history, intent | S | CRAFT | I8 |
| I10 | Documenting the conversion decisions for the next person | S | CRAFT | I9 |

---

## Part J — Operating it afterwards

Conversion isn't finished at cutover. This part is what the other guides never
write.

| ID | Unit | Size | Sourcing | Deps |
| --- | --- | --- | --- | --- |
| J1 | Cost after conversion: what actually changed, and how to see it | M | CRAFT | — |
| J2 | Monitoring incremental drift in production | M | CRAFT | H5 |
| J3 | Scheduled full-refresh reconciliation as a standing control | M | CRAFT | J2 |
| J4 | Alerting: test failures vs run failures | M | CORE | H12 |
| J5 | Ownership and on-call handover | S | CRAFT | I10 |
| J6 | Freshness checks | S | CORE | — |
| J7 | Schema evolution over time | M | SRC | D8 |
| J8 | Growing partition counts and the 4,000 limit | S | CRAFT | D6 |
| J9 | When to revisit the strategy choice | M | CRAFT | J1 |

J3 is the single highest-value operational control in the whole track: a
scheduled full-refresh into a scratch dataset, diffed against production, is the
only reliable detector for the silent failures the other tracks document.

---

## Part K — Anti-patterns

| ID | Unit | Size | Sourcing | Deps |
| --- | --- | --- | --- | --- |
| K1 | The mega-model: one model doing what five should | M | CRAFT | C3 |
| K2 | Hooks as a general-purpose escape hatch | M | CRAFT | F17 |
| K3 | Incremental applied where a table belongs | M | CRAFT | A7 |
| K4 | `run-operation` used as a scheduler | S | CRAFT | C10 |
| K5 | Rebuilding the script's imperative structure in Jinja | M | CRAFT | C6, C7 |
| K6 | Porting the bug faithfully | M | CRAFT | A6 |
| K7 | Over-parameterising with vars | S | CRAFT | G3 |
| K8 | One model per script, mechanically | M | CRAFT | C3 |
| K9 | Ephemeral overuse | S | CRAFT | C2 |
| K10 | No tests, because the script had none | S | CRAFT | H12 |
| K11 | Converting and optimising in the same change | M | CRAFT | H1 |
| K12 | Trusting a green run | S | CRAFT | J2 |

K6 deserves more than it sounds like. Scripts accumulate compensating hacks — a
`DELETE` fixing yesterday's double-insert, a filter working around a duplicate
upstream. Port them literally and you carry the bug forward with a clean face on
it. Conversion is the moment to find these, and [A6](#part-a--assess-before-you-convert)
is what makes it possible.

---

## Delivery waves

138 units won't be written in order. These waves each leave the track coherent
and useful on its own.

### Wave 1 — Foundation (10 units)

Makes the track usable and covers the two highest-risk conversions end to end.

[`E1`](E-translation/E1-one-statement-per-model.md) ✅ ·
[`A3`](A-assess/A3-classify-by-write-pattern.md) ✅ ·
[`A7`](A-assess/A7-what-not-to-convert.md) ✅ ·
[`B8`](B-write-patterns/B8-merge-on-clause-to-unique-key.md) ✅ ·
[`B13`](B-write-patterns/B13-delete-insert-to-insert-overwrite.md) ✅ ·
[`B14`](B-write-patterns/B14-when-the-range-can-empty.md) ✅ ·
[`F17`](F-hooks/F17-when-a-hook-is-wrong.md) ✅ ·
[`F4`](F-hooks/F4-where-hooks-run.md) ✅ ·
[`H2`](H-verification/H2-row-count-parity.md) ✅ ·
[`A9`](A-assess/A9-correctness-baseline.md) ✅

**Waves 1–3 are complete.** Five of the eleven parts are finished: assessment,
write-pattern archetypes, statement-level translation, hooks, and verification.
That is the full path from "what have I got" to "the old script is retired".

**Waves 1–4 are complete.** Seven of the eleven parts are finished. Everything
about *translating a script* is now written — what you have, what shape it is,
how each construct maps, hooks, and how to prove the result.

Remaining is wave 5: Parts G, I, J and K — scheduling and backfills, migration
strategy, operating it afterwards, and anti-patterns.

Rationale: E1 is the constraint everything references. A3 routes readers. B8 and
B13 are the two conversions people actually arrive with. **B14 carries the
behaviour-change warning** and is the reason the track exists. F17 prevents the
most common bad outcome. H2 makes the rest verifiable.

### Wave 2 — Completing the archetypes (26 units)

All of Part B, the Part E units the archetype pages link into, and the rest of
the assessment work.

Remaining `B1`–`B16` (13) · `E2`–`E8` (7) · remaining Part A (6) — **all written ✅**

Part A and Part E are now complete. Part B is complete except `B8`, `B13` and
`B14`, which shipped in wave 1 — so **the whole archetype catalogue is done**.

### Wave 3 — Hooks and verification (27 units)

The two parts where being source-derived is a real differentiator.

Remaining Part F (15) · remaining Part H (12) — **all written ✅**

Parts F and H are now complete.

### Wave 4 — Structure and data movement (33 units)

The long tail of script shapes, plus the statement-level details they need.

Part C (14) · Part D (14) · `E9`–`E13` (5) — **all written ✅**

Parts C, D and E are now complete.

### Wave 5 — Operations and migration (42 units)

Part G (11) · Part I (10) · Part J (9) · Part K (12)

Waves sum to 138.

### dbt-core is now pinned

The blocker is cleared. dbt-core is pinned alongside dbt-adapters in
[the repository README](../../README.md) — `1.latest` @ `300e80c`, released
1.12.3, with released-vs-branch diffing done the same way.

**One trap found in the process, worth recording:** `dbt-labs/dbt-core`'s `main`
branch is no longer the Python implementation. It hosts **dbt Core v2.0, a Rust
rewrite** (the Fusion engine, beta). The Python dbt Core lives on `1.latest`.
Pinning `main` would have documented a different engine.

**Four of the seventeen are already verified** and marked `CORE✓` — ready to
write without further reading:

| Unit | Resolved |
| --- | --- |
| `F6` | `Hook.transaction` defaults to `True`; only explicit `transaction: false` vanishes on BigQuery |
| `F14` | `RunHookType.Start`/`End` = `on-run-start`/`on-run-end` |
| `F16` | *(partly)* microbatch clears `pre_hook` after batch 0 and `post_hook` before the last batch |
| `G6` | Full batching machinery: `BatchSize`, `lookback`, `begin`, boundary maths, per-batch context |

**Thirteen still need reading**, but are no longer blocked — just unstarted:

`C2` · `C13` · `D1` · `E2` · `E3` · `E11` · `G3` · `G4` · `G9` · `G10` · `H12` ·
`J4` · `J6`

`E2` and `E3` remain the ones to do first, since they land in Wave 2.

`E2` and `E3` land in Wave 2, so this blocks early. Two options — pin dbt-core
now, or write those units with their uncertainty stated inline. Worth deciding
deliberately rather than discovering it mid-wave.

---

Back to [the conversion track](README.md) ·
Up to [the repository README](../../README.md)
