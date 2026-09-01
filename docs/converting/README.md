# The conversion track

**Status: planned. This page is the map, not the material.**

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
| `CORE` | Depends on dbt-core behaviour, which has **not** been read. Needs its own pinned commit before being written as fact. |
| `CRAFT` | Judgment and practice. Not derivable from any source. Will be written as advice, and labelled as such. |

Mixing these silently would undermine the thing that makes this repo worth
reading. Pages will say which they are.

## The eight parts

| Part | Subject | Units |
| --- | --- | --- |
| **A** | [Assess before you convert](BACKLOG.md#part-a--assess-before-you-convert) | 5 |
| **B** | [Script archetypes → dbt shapes](BACKLOG.md#part-b--script-archetypes--dbt-shapes) | 16 |
| **C** | [Statement-level translation](BACKLOG.md#part-c--statement-level-translation) | 8 |
| **D** | [Hooks](BACKLOG.md#part-d--hooks) | 10 |
| **E** | [Scheduling, parameters, backfills](BACKLOG.md#part-e--scheduling-parameters-backfills) | 6 |
| **F** | [Proving the conversion is correct](BACKLOG.md#part-f--proving-the-conversion-is-correct) | 6 |
| **G** | [Migration strategy](BACKLOG.md#part-g--migration-strategy) | 5 |
| **H** | [Anti-patterns](BACKLOG.md#part-h--anti-patterns) | 6 |

**62 units total.** Full breakdown, with sizes and dependencies, in
[the backlog](BACKLOG.md).

## Three findings already banked

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

## Where to start

The backlog proposes a [first slice](BACKLOG.md#recommended-first-slice) of eight
units that makes the track useful on its own before the other 54 exist.

---

Back to [the repository README](../../README.md)
