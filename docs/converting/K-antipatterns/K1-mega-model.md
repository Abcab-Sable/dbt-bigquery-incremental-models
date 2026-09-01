# K1 · The mega-model

> **Part K — Anti-patterns** · Sourcing: `CRAFT`
> **The question:** the script was one file, so why isn't one model right?

Because a script's file boundary was about scheduling, not about meaning. Porting
a 400-line script into one model preserves an accident.

## The shape

One model, fifteen CTEs, four hundred lines, producing one table. Every temp table
from the script became a CTE ([C1](../C-structural/C1-multi-statement-to-ctes.md))
and nothing else changed.

It runs. It's also:

- **Untestable in the middle.** You can assert things about the output, nothing
  about the fifteen steps that produced it.
- **Undebuggable.** When [H4](../H-verification/H4-column-level-diffing.md) says a
  column is wrong, you have one place to look and no way to narrow it.
- **Unreusable.** The next model needing "active users" recomputes it.
- **Expensive to iterate.** Every change rebuilds everything.

## The tell

Ask what the model *is*. If the answer needs "and then" more than twice, it's
several models:

> "It cleans events, **and then** joins users, **and then** aggregates daily,
> **and then** ranks by revenue."

Four things. Probably three models and a mart.

## The fix

Split by meaning, not by line count —
[C3](../C-structural/C3-separate-models.md). Each step that produces a nameable
intermediate becomes a model:

```
models/staging/stg_events.sql
models/intermediate/int_events_enriched.sql
models/marts/daily_summary.sql
```

Materialize the layers appropriately: staging as `view` or `ephemeral`,
intermediates as `view` unless expensive, marts as `table` or `incremental`.

Only the mart usually needs to be incremental — making every layer incremental
multiplies the failure modes in this documentation by the number of layers.

## The counter-anti-pattern

Don't over-correct into [K8](K8-one-model-per-statement.md) — one model per script
statement, including the trivial ones. A `select` filtering two rows doesn't need
a name, a file, and a relation.

The test is the same in both directions: **does this intermediate deserve a
name?** Not "is it a separate statement", and not "would splitting be tidier".

## Why conversions produce them

Two reasons, both understandable:

**It's the fastest path.** Wrapping the whole script in CTEs is one file and one
verification exercise. Splitting is more work up front.

**The script's structure feels authoritative.** It ran correctly for years, so
copying its shape feels safe.

But the script's shape was constrained by having no `ref()`. Yours isn't, and
keeping the constraint keeps the costs without the reason.

## The debugging argument

If you're unsure, split. During
[verification](../H-verification/H4-column-level-diffing.md) you'll compare
against the script's intermediate results — and separate models let you compare
stage by stage. One giant model gives you a single diff and no way to bisect it.

That alone usually justifies splitting during a conversion, even if you'd collapse
some of it later.

---

Next: [K2 · Hooks as an escape hatch](K2-hooks-as-escape-hatch.md) ·
Back to [the backlog](../BACKLOG.md)
