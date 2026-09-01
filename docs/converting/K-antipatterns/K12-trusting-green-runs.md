# K12 · Trusting a green run

> **Part K — Anti-patterns** · Sourcing: `CRAFT`
> **The question:** the pipeline is green every morning. Isn't that the point?

A green run means the SQL executed. It does not mean the data is correct. This is
the assumption underneath most of the failures in this documentation, and it's the
last anti-pattern because it's the one that lets the others persist.

## What green actually tells you

A successful `dbt build` means:

- every model's SQL parsed and compiled
- every statement ran without a database error
- every test that exists passed

It does **not** mean:

- the output matches what a full rebuild would produce
- partitions that should be empty are
- no duplicates accumulated
- every column your model produces reached the table
- rows that should have been deleted were

## The failures that stay green

Every one of these succeeds:

| Failure | Why green |
| --- | --- |
| [Empty partition retains stale rows](../B-write-patterns/B14-when-the-range-can-empty.md) | The `MERGE` ran fine; the array was just empty |
| [New column silently dropped](../D-data-movement/D8-add-column-migrations.md) | `dest_columns` came from the target. No error |
| [Composite-key duplicates](../B-write-patterns/B8-merge-on-clause-to-unique-key.md) | Rows inserted successfully |
| [Append-only merge](../../balanced/02-choosing-a-strategy.md#merge-without-a-unique_key-is-append-only) | `on FALSE` is valid SQL |
| [Late data outside the window](../G-scheduling/G8-late-arriving-data.md) | Nothing was asked to process it |
| [Predicate stopped pruning](../J-operating/J1-cost-after-conversion.md) | Correct results, larger bill |

Six ways to be wrong with a green pipeline. None produces a log line.

## Why the instinct is strong

Because for the script, it was nearly true. A `CREATE OR REPLACE` script that
completes has rebuilt the table from source — if the SQL was right, the output is
right.

The instinct is calibrated on a system that couldn't drift. Applied to an
incremental model, it's badly wrong, and nothing in the interface signals the
change.

## What to trust instead

**Tests**, for stated rules — [K10](K10-no-tests.md). `unique` on the key catches
duplicate accumulation on the next run.

**Reconciliation**, for drift — [J3](../J-operating/J3-scheduled-reconciliation.md).
Rebuild in a scratch dataset, diff against production. The only thing that catches
the empty partition.

**Freshness**, for upstream — [J6](../J-operating/J6-freshness-checks.md).
Distinguishes "nothing to process" from "we stopped processing".

**Volume and recency checks**, for the obvious cases —
[J2](../J-operating/J2-monitoring-drift.md).

None of these is expensive. All of them are things the script never needed.

## Put it in the runbook

The sentence worth writing down where whoever inherits this will read it:

> A successful dbt run means the SQL executed. Most incremental failure modes
> produce successful runs. Check the reconciliation output before concluding the
> data is fine.

That paragraph in [I10](../I-migration/I10-documenting-decisions.md) is worth more
than several tests, because it changes what people do when someone asks whether
the numbers are right.

## The one habit

Compare against a full refresh, periodically, automatically. It's the only check
that answers "is this model still correct" rather than "did it run".

Set it up during the conversion, while you still remember what correct looks like.
Everything else in Part J is cheaper and weaker; this is the one that works.

---

Previous: [K11 · Converting and optimising together](K11-convert-and-optimise.md) ·
**End of the conversion track.** Back to [the backlog](../BACKLOG.md) ·
[track index](../README.md)
