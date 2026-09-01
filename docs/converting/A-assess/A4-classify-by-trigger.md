# A4 · Classify by trigger

> **Part A — Assess before you convert** · Sourcing: `CRAFT`
> **The question:** what makes this run, and does dbt have an equivalent?

[A3](A3-classify-by-write-pattern.md) classifies how a script writes. This one
classifies what starts it. The two are independent, and the trigger decides
whether conversion is straightforward or whether something has to stay outside
dbt.

## The four triggers

| Trigger | Signature | Converts how |
| --- | --- | --- |
| **Scheduled** | cron, scheduled query, DAG timer | Directly — [G1](../BACKLOG.md#part-g--scheduling-parameters-backfills) |
| **Chained** | runs after another job succeeds | Becomes a `ref()` edge — [E2](../E-translation/E2-ordering-by-ref.md) |
| **Event-driven** | file lands, message arrives, webhook | **Stays outside.** dbt has no trigger model |
| **Manual** | someone runs it | Depends entirely on why |

## Scheduled

The easy case, and the majority. The only thing to capture carefully is the
**timezone and the offset**, because they decide which partition counts as
"today".

A script running at 23:30 local in a UTC warehouse is writing tomorrow's
partition by UTC reckoning. Get this wrong and every conversion looks off by one
day. Record the schedule in UTC in your [A9](A9-correctness-baseline.md)
baseline.

## Chained

The best conversions in the whole track. "Run B after A" is a dependency someone
expressed as timing because they had no way to express it as a graph. `ref()` is
that way.

Convert both, replace the chain with a `ref()`, and delete the offset. The
ordering becomes enforced rather than hoped for.

## Event-driven

**dbt has no event trigger.** There is no config that means "run when a file
lands". Something outside dbt has to notice the event and invoke it.

That's fine — it just means the trigger stays where it is and calls `dbt build
--select <model>` instead of running your script. The transformation converts;
the trigger doesn't.

Don't try to simulate this by polling inside a model. That's a scheduler written
in SQL, and it's [K4](../BACKLOG.md#part-k--anti-patterns).

## Manual

Ask why, because the answer determines everything:

- **"Whenever we notice it's stale"** ⇒ it wants a schedule. Converting improves
  it.
- **"After the finance team confirms"** ⇒ a genuine human gate. Keep the gate,
  convert the transformation.
- **"When we're backfilling"** ⇒ not a trigger, a mode. Becomes
  [G5–G7](../BACKLOG.md#part-g--scheduling-parameters-backfills).
- **"It was a one-off, it just never got deleted"** ⇒ don't convert it. Delete
  it, or leave it — [A7](A7-what-not-to-convert.md).

That last answer is more common than anyone admits, and it's the cheapest
possible outcome of this exercise.

## The consolidation opportunity

Once triggers are classified, look at the scheduled ones together. Twelve
scripts on twelve staggered crons, each guessing at the previous one's runtime,
usually collapse into **one `dbt build`** whose ordering is derived rather than
guessed.

That consolidation is often the single largest operational win of a migration,
and you can only see it once the inventory and the triggers are both written
down. Details in [G2](../BACKLOG.md#part-g--scheduling-parameters-backfills).

---

Previous: [A2 · Map dependencies](A2-map-dependencies.md) ·
Next: [A5 · Find the hidden state](A5-hidden-state.md)
