# H11 · Differences that are supposed to exist

> **Part H — Proving correctness** · Sourcing: `CRAFT`
> **The question:** the new output is different, and I think that's right. Now what?

Predict it, quantify it, and get it agreed — before you show anyone the diff.
A difference you predicted is evidence the conversion is understood. The same
difference discovered during review is a bug until proven otherwise.

## Where legitimate differences come from

**A compensating hack you dropped.** [A6](../A-assess/A6-compensating-hacks.md).
A stale dedup removed means rows reappear; a lifted vendor exclusion means rows
arrive. Both are intended.

**The old script wasn't idempotent.** If someone re-ran it in 2024, the baseline
holds duplicates your correct model won't reproduce —
[E7](../E-translation/E7-idempotency-meaning.md).

**Known-bad historical data** the script never fixed. Your model may compute it
correctly, so the periods legitimately disagree.

**A behaviour change you chose.** The clearest case is
[B14](../B-write-patterns/B14-when-the-range-can-empty.md): if the script left
empty periods populated and your model clears them, that period now differs — and
that was the point.

**Fixed null handling or types.** [H8](H8-reconciling-nulls.md),
[H9](H9-reconciling-numeric-precision.md). Right, and different.

## Predict before you compare

The order matters more than it sounds. Write the expected differences down
*before* running the parity check:

```
Expected differences (written 2026-09-01, before first comparison):

1. 2023-03-01 → 2023-06-30: ~40 extra rows/day.
   Cause: dropped `qualify row_number()` that deduped a 2023 double-load.
   Source is clean now (verified). Extra rows are legitimate.
   Confirmed with: Priya, 2026-08-29.

2. 2026-08-14: entire partition differs.
   Cause: known-bad upstream. Old script wrote garbage; new model produces
   nothing (correct — see A9 baseline).

3. All partitions: `status` null where old had ''.
   Cause: removed COALESCE(status,'') — see H8.
   DOWNSTREAM IMPACT: `where status != 'cancelled'` now excludes these rows.
   Notified: reporting team, 2026-08-30.
```

Anything not on that list, appearing in the diff, is a conversion bug. That's the
value — it converts an open-ended investigation into a closed question.

## Quantify each one

"Some extra rows" isn't checkable. Bound it:

```sql
select
    event_date,
    old.n as old_rows,
    new.n as new_rows,
    new.n - old.n as delta
from (select event_date, count(*) n from analytics.daily_events group by 1) old
full outer join (select event_date, count(*) n from analytics_dbt.daily_events group by 1) new
using (event_date)
where old.n is distinct from new.n
order by event_date;
```

Then check the observed delta matches the predicted one. "I expected ~40/day and
saw 38–43" is a verified prediction. "I expected more rows and there are more
rows" is not.

## Get the impactful ones agreed

Differences with downstream consequences need a named person to accept them
before cutover, not after:

| Difference | Needs agreement from |
| --- | --- |
| Row counts change materially | Whoever reports on the numbers |
| Null semantics change | Anyone filtering that column |
| A period becomes empty | Whoever expected data there |
| Types change | Anyone joining or casting |

Record the name and date. During cutover, "this was signed off" is the difference
between a planned change and an incident —
[I5](../BACKLOG.md#part-i--migration-strategy).

## When you can't explain one

Stop. An unexplained difference is a conversion bug until proven otherwise, and
the temptation at this point — usually late in the process, with pressure to
finish — is to call it noise.

Work through [H4](H4-column-level-diffing.md) to find the column, then
[H7](H7-reconciling-ordering.md)–[H10](H10-reconciling-timestamps.md) for the
usual mechanical causes. If it survives all of those, it's real.

## The rule

**Every difference is either predicted or a bug.** There is no third category, and
"probably fine" is the phrase that precedes most bad cutovers.

---

Previous: [H10 · Reconciling: timestamps and timezones](H10-reconciling-timestamps.md) ·
Next: [H12 · Tests from the script's guarantees](H12-tests-from-guarantees.md)
