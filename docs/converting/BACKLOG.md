# Conversion track — backlog

Hyper-granular decomposition of [the conversion track](README.md). Each unit is
sized to be written in one sitting and to stand alone as a page or a section.

## Legend

**Status** — `todo` · `drafting` · `review` · `done`

**Size** — `S` short section · `M` full page · `L` long page, split if it grows

**Sourcing** — `SRC` verifiable against dbt-adapters at the
[pinned commit](../../README.md) · `CORE` depends on unread dbt-core behaviour ·
`CRAFT` judgment and practice

Every unit is `todo`. The column is there so it stays useful as we work.

---

## Part A — Assess before you convert

Nothing here produces a model. It stops you converting the wrong thing, or
converting the right thing without knowing what "correct" meant.

| ID | Unit | Size | Sourcing |
| --- | --- | --- | --- |
| A1 | Inventory your scripts: type, schedule, owner, declared and undeclared dependencies | S | CRAFT |
| A2 | Deciding what **not** to convert — scripts that should stay scripts | S | CRAFT |
| A3 | Classifying a script by write pattern: replace / append / upsert / delete-insert / DDL / side-effect | M | CRAFT |
| A4 | Finding the hidden state: manual steps, out-of-band fixes, things the script assumes already exist | M | CRAFT |
| A5 | Establishing a correctness baseline **before** you touch anything | M | CRAFT |

A3 is the spine of Part B — the classification decides which archetype page
applies. A5 pairs with Part F; capture the baseline early or you have nothing to
prove against later.

---

## Part B — Script archetypes → dbt shapes

The catalogue. Each unit: the script pattern, the dbt shape, a worked
before/after, and what breaks in translation.

| ID | Unit | Size | Sourcing |
| --- | --- | --- | --- |
| B1 | `CREATE OR REPLACE TABLE x AS SELECT` → `materialized='table'` | S | SRC |
| B2 | `CREATE VIEW` → `materialized='view'`, and when a view should become a table | S | SRC |
| B3 | Unfiltered `INSERT INTO ... SELECT` → incremental with append semantics | M | SRC |
| B4 | `INSERT INTO ... SELECT` with a watermark filter → incremental + `is_incremental()` | M | SRC |
| B5 | Hand-written `MERGE` → incremental + `unique_key`, mapping clause by clause | L | SRC |
| B6 | `DELETE` + `INSERT` for a date range → `insert_overwrite` | L | SRC |
| B7 | `TRUNCATE` + `INSERT` → `table`, or `insert_overwrite` when partitioned | M | SRC |
| B8 | Multi-statement script with temp tables → CTEs vs ephemeral vs separate models | L | CRAFT |
| B9 | Stored procedure with `DECLARE`/`SET` scalars → vars, macros, `_dbt_max_partition` | L | SRC |
| B10 | Stored procedure with `IF`/`WHILE`/`LOOP` → why this mostly can't map, and what to do instead | L | CRAFT |
| B11 | `EXECUTE IMMEDIATE` and dynamic SQL → macros, or a deliberate `run-operation` | M | CRAFT |
| B12 | Scheduled query with `@run_date`-style parameters → vars, or microbatch `event_time` | M | CORE |
| B13 | Python script using the BigQuery client → Python model, SQL + seed, or stays external | L | CORE |
| B14 | Shell / `bq` CLI orchestration → project structure and the `ref()` DAG | M | CRAFT |
| B15 | Airflow DAG of BigQuery operators → dbt DAG, and what legitimately stays in Airflow | M | CRAFT |
| B16 | DDL scripts: partitioning, clustering, expiration, labels, description → configs vs hooks | M | SRC |

B5 and B6 are the two that matter most — they're where hand-rolled incremental
logic actually lives, and where the existing tracks' findings become directly
load-bearing. B6 must link the [empty-partition trap](../balanced/04-insert-overwrite.md#the-empty-partition-trap):
a `DELETE`+`INSERT` script *does* empty a range, and the naive
`insert_overwrite` conversion does not. **That's a behaviour change introduced by
the conversion**, and it is the single most important thing this track has to
say.

B10 needs an honest answer rather than a clever one. Most procedural loops are
either a set-based operation in disguise or genuine orchestration that belongs
outside dbt.

---

## Part C — Statement-level translation

The mechanical problems that appear inside almost every conversion, factored out
so Part B can link to them instead of repeating.

| ID | Unit | Size | Sourcing |
| --- | --- | --- | --- |
| C1 | One statement per model: the constraint everything else follows from | S | SRC |
| C2 | Ordering by `ref()` instead of by line number | M | CORE |
| C3 | Idempotency: making a re-run safe, and proving it | M | CRAFT |
| C4 | Where hardcoded table names go: `ref` vs `source`, and never both | S | CORE |
| C5 | Cross-project and cross-dataset references | S | CRAFT |
| C6 | Session settings and `DECLARE`d values that spanned statements | M | SRC |
| C7 | UDFs: persistent vs temporary, and where they live in a dbt project | M | CRAFT |
| C8 | Hardcoded dates and manual backfill parameters | S | CRAFT |

C1 is the conceptual hinge of the whole track. A script is a sequence of
statements; a model is one `select`. Almost every "how do I do X in dbt" question
during a conversion is a consequence of that single constraint, and stating it
early saves repeating it sixteen times.

---

## Part D — Hooks

Deep, because hooks are where conversions go to die. A script's leftover
statements get swept into a `post_hook` and the model quietly becomes a script
again.

Three facts here are already source-verified — see
[the track index](README.md#three-findings-already-banked).

| ID | Unit | Size | Sourcing |
| --- | --- | --- | --- |
| D1 | What a hook is: `run_hooks` mechanics, Jinja rendering, empty-hook skipping | M | SRC |
| D2 | Exactly where hooks run in the BigQuery incremental materialization | M | SRC |
| D3 | The `transaction` filter, and why `transaction: false` never fires on BigQuery | M | SRC + CORE |
| D4 | pre-hook patterns worth keeping | M | CRAFT |
| D5 | post-hook patterns worth keeping | M | CRAFT |
| D6 | Grants: the `grants` config vs a post-hook, and the ordering trap | M | SRC |
| D7 | Table options via post-hook: expiration, labels, description | S | CRAFT |
| D8 | Audit and logging rows: post-hook vs `on-run-end` | M | CRAFT |
| D9 | `on-run-start` / `on-run-end` vs per-model hooks | M | CORE |
| D10 | When a hook is the wrong answer | M | CRAFT |

D3 needs care and is the reason that row is tagged twice. The **filtering**
behaviour is verified in dbt-adapters: `run_hooks` does
`selectattr('transaction', 'equalto', inside_transaction)`, and every BigQuery
materialization calls it only at the `True` default. What is **not** verified is
the default `transaction` value dbt-core assigns to a plain-string hook. If that
default is `True`, ordinary hooks are unaffected and only explicit
`transaction: false` silently vanishes. **Read dbt-core and pin it before writing
this unit as fact.**

D6's trap is source-verified: post-hooks run *before* `apply_grants`, so a
post-hook granting access races the `grants` config rather than composing with
it.

D10 is the most valuable unit in Part D and should be written early. Everything
else in the part explains how hooks work; this one explains why the answer is
usually a model.

---

## Part E — Scheduling, parameters, backfills

| ID | Unit | Size | Sourcing |
| --- | --- | --- | --- |
| E1 | From cron entries to `dbt build` scheduling | M | CRAFT |
| E2 | Passing dates in: `vars`, `--vars`, environment variables | M | CORE |
| E3 | Backfilling a freshly converted model | L | CORE |
| E4 | Late-arriving data after conversion | M | CRAFT |
| E5 | Partial runs: `--select`, `--exclude`, and state-based selection | M | CORE |
| E6 | Retry and failure semantics vs a script that half-completed | M | CRAFT |

E6 is underrated. A failed script often leaves partial writes; a failed dbt run
has different guarantees depending on materialization. Converting changes your
failure modes, and nobody documents that.

E3 should lean on [microbatch](../balanced/05-microbatch.md) while repeating its
caveat that the batching machinery is dbt-core and unverified here.

---

## Part F — Proving the conversion is correct

The part everyone skips and everyone should not. A conversion isn't done when the
model runs; it's done when you can show it matches.

| ID | Unit | Size | Sourcing |
| --- | --- | --- | --- |
| F1 | Row-count and checksum parity between old and new | M | CRAFT |
| F2 | Shadow mode: running both in parallel | M | CRAFT |
| F3 | Diffing outputs, including column-level | L | CRAFT |
| F4 | Reconciling legitimate differences: ordering, nulls, floats, timestamp precision | M | CRAFT |
| F5 | dbt tests that encode what the script implicitly guaranteed | M | CRAFT |
| F6 | Sign-off criteria, and when it's safe to delete the old script | S | CRAFT |

F4 is where conversions actually stall. Differences are usually legitimate —
float summation order, timestamp precision, null-vs-empty-string — and without a
framework for triaging them people either chase noise for a week or wave through
a real bug.

F5 connects to the beginner track's point that
[a green run is not evidence of correct data](../beginner/06-when-things-go-wrong.md).

---

## Part G — Migration strategy

| ID | Unit | Size | Sourcing |
| --- | --- | --- | --- |
| G1 | Conversion order: leaves first or roots first | M | CRAFT |
| G2 | The strangler pattern applied to a script suite | M | CRAFT |
| G3 | Downstream consumers during cutover | M | CRAFT |
| G4 | Keeping the old script runnable as a rollback | S | CRAFT |
| G5 | Decommissioning: what to delete, what to keep, what to document | S | CRAFT |

---

## Part H — Anti-patterns

| ID | Unit | Size | Sourcing |
| --- | --- | --- | --- |
| H1 | The mega-model: one model doing what five should | M | CRAFT |
| H2 | Hooks as a general-purpose escape hatch | M | CRAFT |
| H3 | Incremental applied to something that should be a table | M | CRAFT |
| H4 | `run-operation` used as a scheduler | S | CRAFT |
| H5 | Rebuilding the script's imperative structure in Jinja | M | CRAFT |
| H6 | Converting a broken script faithfully, and porting the bug | M | CRAFT |

H6 deserves more than it sounds like. Scripts accumulate compensating hacks —
a `DELETE` that fixes yesterday's double-insert, a filter working around a
duplicate upstream. Port them literally and you carry the bug forward with a
clean face on it. The conversion is the moment to find these, and the
[baseline from A5](#part-a--assess-before-you-convert) is what lets you tell a
bug from a feature.

---

## Recommended first slice

Eight units that make the track independently useful before the other 54 exist.
Chosen so the two highest-risk conversions — hand-written `MERGE` and
`DELETE`+`INSERT` — are covered end to end, with the reasoning and the
verification either side of them.

| Order | ID | Why first |
| --- | --- | --- |
| 1 | **C1** | One statement per model. Everything else references it. |
| 2 | **A3** | Classification. Decides which archetype page a reader needs. |
| 3 | **B5** | Hand-written `MERGE` → `unique_key`. The most common real conversion. |
| 4 | **B6** | `DELETE`+`INSERT` → `insert_overwrite`. Carries the behaviour-change warning. |
| 5 | **D10** | When a hook is the wrong answer. Prevents the most common bad outcome. |
| 6 | **D2** | Where hooks actually run. Source-verified, and D4–D8 all depend on it. |
| 7 | **F1** | Parity checking. Makes 3 and 4 verifiable rather than hopeful. |
| 8 | **A2** | What not to convert. Cheap, short, and saves the most total effort. |

Roughly one page each, so a track of eight pages that already answers the
questions people actually arrive with.

**Before D3 or any `CORE` unit is written as fact**, dbt-core needs reading and
pinning the way dbt-adapters already is. Until then those units stay `todo`, or
get written with their uncertainty stated explicitly. Which of those two we do is
worth deciding deliberately rather than by accident.

---

Back to [the conversion track](README.md) ·
Up to [the repository README](../../README.md)
