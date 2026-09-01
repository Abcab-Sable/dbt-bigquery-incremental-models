# K8 · One model per script statement, mechanically

> **Part K — Anti-patterns** · Sourcing: `CRAFT`
> **The question:** I split the script into models. Isn't more granular better?

Not mechanically. The opposite error to [K1](K1-mega-model.md), and it comes from
the same mistake — treating the script's structure as meaningful rather than
incidental.

## The shape

A twelve-statement script becomes twelve models:

```
models/int_events_step_01.sql   -- select from raw, rename two columns
models/int_events_step_02.sql   -- filter out nulls
models/int_events_step_03.sql   -- cast a column
models/int_events_step_04.sql   -- join users
...
models/daily_summary.sql
```

Each one runs, each has lineage. And the project now has twelve relations where
two would do, named after positions in a file that no longer exists.

## What's wrong

**The names carry no meaning.** `int_events_step_03` tells a reader nothing. In
six months nobody knows which step does what without opening all twelve.

**Materialization cost.** Every non-ephemeral model is a table or view. Twelve
objects to store, refresh and reason about, for one output.

**Slower.** Twelve separate queries, each reading the previous one's output, where
BigQuery could have optimised one query with CTEs.

**Harder to change.** A logic change spanning three steps means editing three
files and checking the joins between them.

**The DAG is noise.** A lineage graph of twelve linear nodes tells you nothing a
single node wouldn't.

## The test, same as K1

**Does this intermediate deserve a name?**

A model earns one when it's reused, expensive, testable, or a business concept
someone would say out loud. "Active users" earns a name. "The bit after we cast
the timestamp" does not.

Everything failing that test is a CTE —
[C1](../C-structural/C1-multi-statement-to-ctes.md).

## The right granularity

The same twelve statements usually collapse to:

```
models/staging/stg_events.sql          -- steps 1–4: clean and standardise
models/intermediate/int_events_enriched.sql  -- steps 5–8: join dimensions
models/marts/daily_summary.sql         -- steps 9–12: aggregate
```

Three models, each a nameable thing, each with CTEs inside doing the sub-steps.
Testable at the boundaries that matter, debuggable stage by stage, and readable.

## Why it happens

Usually over-correction. Someone reads [K1](K1-mega-model.md), decides mega-models
are bad, and splits maximally. Or the conversion is done mechanically —
statement in, model out — because that's easy to automate.

Both produce a project whose shape reflects the old script rather than the data.

## Naming is the signal

If you're struggling to name a model, that's the answer. `int_events_step_03`
isn't a name, it's an admission. `stg_events_deduplicated` is a name — and if the
step deserves it, keep the model.

Rename during conversion, don't preserve positional names. The script's ordering
is exactly what you're replacing.

## The middle ground

If you genuinely want stage-by-stage visibility during
[verification](../H-verification/H4-column-level-diffing.md) — comparing each
stage against the script's temp tables — split more finely *temporarily*, then
collapse once parity is proven.

Legitimate, as long as collapsing actually happens. Put it in the ticket, or it
won't.

---

Previous: [K7 · Over-parameterising with vars](K7-over-parameterising.md) ·
Next: [K9 · Ephemeral overuse](K9-ephemeral-overuse.md)
