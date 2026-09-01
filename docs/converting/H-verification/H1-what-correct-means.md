# H1 · What "correct" means for this conversion

> **Part H — Proving correctness** · Sourcing: `CRAFT`
> **The question:** what am I actually trying to prove?

Decide this before running any comparison, because "the same" has at least four
meanings and they demand different evidence.

## Four definitions, in increasing strength

**1. Same row count.** Cheapest, weakest. Catches duplication and missing
partitions. Proves nothing about values. — [H2](H2-row-count-parity.md)

**2. Same rows.** Every row in old is in new and vice versa. Catches value
errors. Needs a stable comparison key. — [H3](H3-checksum-parity.md)

**3. Same rows, same values, every column.** The strongest practical bar, and the
one that surfaces the differences nobody predicted. — [H4](H4-column-level-diffing.md)

**4. Same behaviour over time.** Not a snapshot at all — the two produce the same
output across a period, including edge days. — [H5](H5-shadow-mode.md)

Most conversions need 3 for a sample and 4 for a week. Reaching for 4 without
having done 3 means you're comparing two things you don't understand yet.

## The uncomfortable question

**Was the old output correct?**

You are proving *equivalence*, not correctness. If the script had a bug, a
perfect conversion reproduces it, and your parity check passes.

So before comparing, be explicit about which you want:

| Goal | Then |
| --- | --- |
| Equivalence — match the script exactly | Port compensating hacks as-is. Differences are conversion bugs |
| Correctness — fix known problems too | Expect differences. You must be able to explain each — [H11](H11-differences-that-should-exist.md) |

**Pick one.** Doing both at once is [K11](../K-antipatterns/K11-convert-and-optimise.md) and
makes every diff ambiguous: you can't tell a bug from an intended fix.

The default should be equivalence. Convert first, fix second, as two changes.

## Where the answer comes from

From [A9](../A-assess/A9-correctness-baseline.md), and specifically from these:

- **Known-bad data.** If 2024 has three broken days, your new model may fix them
  or reproduce them. Either is fine; not knowing is not.
- **Compensating hacks** ([A6](../A-assess/A6-compensating-hacks.md)). Dropping a
  stale dedup produces a legitimate diff you should predict, not discover.
- **Was the script idempotent?** If not, the baseline may contain duplicates a
  correct model won't reproduce.

If you skipped Part A, you're now reverse-engineering intent from a diff, which is
several times more expensive.

## Scope it

Full-history parity is usually not the goal and often not achievable — upstream
data mutates, and the old script may have processed it under different conditions.

State the scope explicitly:

```
Comparing: analytics.daily_events vs analytics_dbt.daily_events
Period:    2026-08-01 to 2026-08-31 (31 partitions)
Bar:       column-level parity (definition 3), all columns
Excluded:  _loaded_at (non-deterministic by design)
Known:     2026-08-14 known bad upstream, expect mismatch both sides
Goal:      EQUIVALENCE — compensating hacks ported as-is
```

That paragraph is the deliverable of this unit. Everything in H2–H13 is executing
against it, and [H13](H13-sign-off.md) signs it off.

## What can't be proven this way

- **Future behaviour.** Parity today says nothing about next month —
  [J3](../J-operating/J3-scheduled-reconciliation.md) is the standing control.
- **Empty-period behaviour**, unless you force it. Normal data never contains the
  case — [B14](../B-write-patterns/B14-when-the-range-can-empty.md).
- **Late-arriving data**, which by definition hasn't arrived.

Test the first two deliberately. The third needs shadow running.

---

Previous: [F16 · Hooks and failure semantics](../F-hooks/F16-hooks-and-failure.md) ·
Next: [H3 · Checksum and hash parity](H3-checksum-parity.md)
