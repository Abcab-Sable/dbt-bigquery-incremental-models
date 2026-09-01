# A7 · Decide what not to convert

> **Part A — Assess before you convert** · Sourcing: `CRAFT`
> **The question:** should this script become a dbt model at all?

Converting everything is a common and expensive mistake. This page is the
counterweight, and it's short on purpose.

## The trade you're making

A working script has one useful property: it does what it says, in order, and
when it breaks it usually breaks loudly.

Converting buys you lineage, testing, documentation, environment separation, and
a dependency graph that doesn't live in someone's head. Those are real.

It also buys you a class of failure the script didn't have. Every silent failure
documented in [the balanced track](../../balanced/08-gotchas.md) — the partition
that won't empty, the column that vanishes, the duplicate that accumulates —
arrives *with* the conversion. A script that ran a `DELETE` and an `INSERT` had
no empty-partition trap, because it deleted unconditionally.

**Conversion is worth it when the lineage and testing outweigh the new failure
surface.** Often true. Not always.

## Leave it alone if

**It isn't a transformation.** If the output isn't a table or view derived from
other tables, dbt has nothing to materialize. Exports, alerts, API calls, file
moves, permission sweeps — these are jobs, not models.

**It runs once.** Migrations, backfills, one-off corrections. A model implies a
thing that gets rebuilt. If the answer to "what happens when this runs again
tomorrow" is "nothing should", it isn't a model. A `run-operation` or a plain
script is the honest home.

**It's administrative.** Creating datasets, rotating service accounts, managing
quotas, setting org policy. dbt can technically do some of this through hooks.
That doesn't make it the right place.

**It's cheap and correct.** A script that rebuilds a 200 MB table in twelve
seconds for a fraction of a penny is not a cost problem. Converting it to an
incremental model to "do it properly" trades a boring correct thing for an
interesting fragile one. Make it `materialized='table'` if you want it in the
project — but don't make it incremental.

**Its logic is genuinely procedural.** Loops with real dependencies between
iterations, recursive processing, statefulness that isn't expressible as a set
operation. You can force these into Jinja. The result is
[K5](../BACKLOG.md#part-k--anti-patterns), and it's worse than the script was.

**Nobody knows what it does and it isn't breaking.** Harsh but practical. Without
someone who can say what correct output looks like, you cannot do
[Part H](../BACKLOG.md#part-h--proving-correctness), which means you cannot know
whether your conversion worked. Find the owner first, or leave it.

## Convert it if

- Other things depend on its output, and that dependency is currently invisible
- It's expensive, and the cost scales with history rather than with new data
- It's been silently wrong before, or nobody would notice if it were
- It exists in three near-identical copies for dev, staging, and prod
- Its logic needs to be readable by people who won't read SQL scripts

Two or more of those and it's usually worth doing.

## The middle ground

You don't have to choose between "dbt model" and "untouched".

**Source it without converting it.** Declare the script's output table as a dbt
`source`. Downstream models get lineage and freshness checks; the script keeps
running. This is the highest-value, lowest-risk move available, and it's
underused.

**Convert the transformation, keep the side effects.** Most scripts are one
`select` wearing a lot of operational clothing. Model the query; leave the export
and the notification where they are.

**Convert to `table` first, incremental later.** Two separate changes with two
separate risk profiles. Doing them together makes a failure hard to attribute —
[K11](../BACKLOG.md#part-k--anti-patterns).

## Write down what you skipped

For anything you decide not to convert, record the reason next to the
classification from [A3](A3-classify-by-write-pattern.md):

```
gcs_export_daily.sql
  class:    side-effect (EXPORT DATA, no table output)
  decision: not converting — not a transformation
  revisit:  only if the query feeding it becomes shared
```

Otherwise someone re-litigates it in six months, or worse, converts it.

---

Previous: [A3 · Classify by write pattern](A3-classify-by-write-pattern.md) ·
Next: [B8 · The `MERGE` `ON` clause → `unique_key`](../B-write-patterns/B8-merge-on-clause-to-unique-key.md)
