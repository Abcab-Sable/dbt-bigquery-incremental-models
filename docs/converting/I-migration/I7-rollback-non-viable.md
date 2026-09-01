# I7 · When rollback stops being viable

> **Part I — Migration strategy** · Sourcing: `CRAFT`
> **The question:** at what point can we no longer go back?

When reverting would create more inconsistency than it fixes. Know where that line
is, because after it the only direction is forward — and that changes how you'd
respond to a problem.

## The things that cross the line

**Downstream tables built from the new output.** Marts, aggregates, exports. Going
back means rebuilding all of them too, and any that can't be rebuilt from source
are stuck.

**Data the old script can't reproduce.** If the model backfilled history the
script never covered, or fixed rows the script would recompute as wrong, reverting
loses that.

**Schema changes consumers adopted.** A new column people now select. Rolling back
removes it and breaks them — the opposite failure from the one you were fixing.

**A structural change.** Sharded tables consolidated
([D5](../D-data-movement/D5-sharded-tables.md)), a repartitioned table
([D6](../D-data-movement/D6-partitioning-ddl.md)). The script writes to a shape
that no longer exists.

**External delivery.** Files exported, partners notified, downstream systems
loaded. You can't un-send those —
[D2](../D-data-movement/D2-export-data.md).

**The rollback path itself decayed.** Credentials rotated, a referenced table
dropped, the scheduled query deleted by a cleanup. This one happens quietly and is
why [I6](I6-rollback-keeping-script.md) says to test it.

## Say when the window closes

Decide the date at cutover, not by drift:

```
Rollback window: 2026-09-09 → 2026-09-30
Closes after: September month-end close completes
After that:   forward-fix only. Baseline snapshot retained until 2026-12-01.
```

Naming a date forces the question "will we know by then?", which is the useful
part. And it stops the situation where everyone assumes rollback is available and
nobody has checked for six weeks.

## Forward-fix instead

Past the line, the options are:

**Fix the model and rebuild.** Usually right. You have the model, tests and the
parity machinery from Part H — this is the case they were built for.

**Restore from the baseline snapshot**, then run the corrected model forward. Why
[A9](../A-assess/A9-correctness-baseline.md) says to snapshot to a real table
rather than trusting seven days of time travel.

**Accept and document.** If the difference is small and the fix is expensive,
sometimes the honest answer is to record it and move on. Note it where the next
person will find it.

## Reduce what crosses the line

Some things are choices:

- **Don't convert a chain in one go.** Each conversion cut over separately keeps
  the blast radius small — [I1](I1-conversion-order.md).
- **Don't couple structural changes to conversions.** Repartitioning is a separate
  change with its own rollback story — [K11](../K-antipatterns/K11-convert-and-optimise.md).
- **Hold external delivery back.** Keep exports pointed at the old table for the
  first week; they're the least reversible thing you do.
- **Keep the baseline snapshot longer than the rollback window.** Storage is
  cheap; regret isn't.

## Write down where you are

The rollback plan should say which side of the line the conversion currently sits
on:

```
2026-09-09  cut over. Rollback: re-enable script (tested).
2026-09-12  daily_summary now reads the new model. Rollback now requires
            rebuilding daily_summary too — still viable, ~20 min.
2026-09-16  partner export switched to new table. Files delivered.
            ROLLBACK NO LONGER VIABLE — forward-fix only.
```

Three lines in a ticket. It means that when something surfaces on the 20th, the
person responding knows immediately what their options are instead of finding out
the hard way.

---

Previous: [I6 · Rollback: keeping the old script runnable](I6-rollback-keeping-script.md) ·
Next: [I8 · Decommissioning checklist](I8-decommissioning.md)
