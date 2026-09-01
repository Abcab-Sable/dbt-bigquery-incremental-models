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
