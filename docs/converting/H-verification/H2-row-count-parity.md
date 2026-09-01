# H2 · Row-count parity

> **Part H — Proving correctness** · Sourcing: `CRAFT`
> **The question:** cheapest useful check that my conversion matches?

Row counts. Weak evidence, near-zero cost, and they catch the failure modes this
track is actually about. Start here, then move to
[H3](../H-verification/H3-checksum-parity.md) for content parity.

## Total count first

Point both versions at the same input, then:

```sql
select
    (select count(*) from analytics.daily_events)     as old_rows,
    (select count(*) from analytics_dbt.daily_events) as new_rows
```

Equal is necessary, not sufficient. Unequal means stop.

## Per-partition count is the one that matters

The total can match while individual partitions are wrong — one gained what
another lost, which is exactly what a botched `insert_overwrite` conversion looks
like.

```sql
with old as (
    select event_date, count(*) as n
    from analytics.daily_events
    group by event_date
),
new as (
    select event_date, count(*) as n
    from analytics_dbt.daily_events
    group by event_date
)
select
    coalesce(old.event_date, new.event_date) as event_date,
    old.n as old_rows,
    new.n as new_rows,
    coalesce(new.n, 0) - coalesce(old.n, 0) as delta
from old
full outer join new using (event_date)
where old.n is null
   or new.n is null
   or old.n != new.n
order by event_date
```

Empty result set means per-partition parity. Anything else is your bug list, in
date order.

**Run this one even if the totals match.** It's the check that finds
[B14](../B-write-patterns/B14-when-the-range-can-empty.md).

## Reading a mismatch

| Shape | Likely cause |
| --- | --- |
| New has rows where old has none | The old script never wrote that period; check your `is_incremental()` range |
| **Old has rows where new has none** | Expected if you fixed a bug — otherwise your filter is too narrow |
| **New has rows in a period that should be empty** | [B14](../B-write-patterns/B14-when-the-range-can-empty.md). The empty partition wasn't cleared |
| New consistently ~2× old | Append semantics — missing `unique_key`, or a re-run duplicating |
| New slightly higher, scattered | Duplicates from nullable composite key columns |
| Only the newest partition differs | Usually a boundary, not a bug — the two ran at different times |
| Only the oldest partition differs | Watermark or `begin` mismatch |

The bolded rows are conversion-introduced. The others are usually configuration.

## Test the empty case deliberately

Parity on normal data won't reveal the empty-partition problem, because normal
data never has an empty partition. You have to force it:

1. Note the count for a partition that currently has rows.
2. Add a temporary filter making the model produce zero rows for it.
3. Run.
4. Count that partition again.

Old script's behaviour: **0**. Dynamic `insert_overwrite`: **unchanged**.

If those differ, that's the conversion changing semantics, and
[B14](../B-write-patterns/B14-when-the-range-can-empty.md) has the fix.

## Prove idempotency while you're here

Run the new model twice without changing input, counting in between:

```sql
select count(*) from analytics_dbt.daily_events;
```

The two numbers must be identical. If the second is higher, the model appends
where you meant it to replace — almost always a missing `unique_key`. This is one
query and it catches the single most common conversion bug.

## What parity does not prove

Be clear with yourself about the limits:

- **Nothing about column values.** Every count can match with every `amount`
  wrong. That's [H3](../H-verification/H3-checksum-parity.md) and
  [H4](../H-verification/H4-column-level-diffing.md).
- **Nothing about the periods you didn't test.** One good day is one good day.
- **Nothing about late-arriving data**, which by definition hasn't arrived.
- **Nothing about the future.** Parity today is not a standing guarantee, which
  is why [J3](../J-operating/J3-scheduled-reconciliation.md) exists.

Row-count parity is the check that tells you whether it's worth doing the
expensive checks. Passing it is permission to continue, not a result.

## When to run it

- After the first successful build of the converted model
- After changing strategy, `unique_key`, or `partition_by`
- Every day of shadow running ([H5](../H-verification/H5-shadow-mode.md))
- Once more immediately before retiring the script

---

Previous: [F4 · Exactly where hooks run](../F-hooks/F4-where-hooks-run.md) ·
Next in wave 1: [A9 · Capture the correctness baseline](../A-assess/A9-correctness-baseline.md)
