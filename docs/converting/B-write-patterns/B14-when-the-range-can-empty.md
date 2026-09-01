# B14 · When the range can legitimately empty

> **Part B — Write-pattern archetypes** · Sourcing: `SRC`
> **The question:** my `DELETE`+`INSERT` conversion runs clean. Why is there stale data in the table?

This is the most important page in the track. The conversion in
[B13](B13-delete-insert-to-insert-overwrite.md) introduces a bug that your script
did not have, and nothing in the run output tells you.

## The difference in one line

Your script deleted a range **you declared**. dbt deletes a range **derived from
the rows your model produced**.

When your model produces rows for every period, those are the same thing. When it
doesn't, they aren't.

## Watch it happen

An events model, `insert_overwrite`, partitioned by day, rebuilding the last
three days.

**Monday.** Upstream has 1,000 events for `2026-08-31`. The model produces 1,000
rows. dbt collects `{2026-08-31}`, clears that partition, writes 1,000 rows.
Correct.

**Tuesday.** Upstream discovers Monday's feed was corrupt and retracts it.
`raw.events` now has nothing for `2026-08-31`. Your model, correctly, produces
**zero rows** for that day.

Here is what dbt does with that:

```sql
set (dbt_partitions_for_replacement) = (
    select as struct array_agg(distinct event_date IGNORE NULLS) from <tmp>
);
```

No rows in `<tmp>` for `2026-08-31` ⇒ that date isn't in the array ⇒ the
`when not matched by source ... then delete` clause matches nothing for it ⇒
**the 1,000 stale rows stay.**

The run succeeds. Row counts look plausible. No warning is logged. You find out
when someone asks why the dashboard disagrees with source.

**What your script would have done:** `DELETE FROM analytics.daily_events WHERE
event_date >= start_date` removed those rows unconditionally, then inserted
nothing. The table would have been correct.

You didn't convert a script. You changed its semantics.

## Are you exposed?

One question: **can any period your model covers legitimately produce zero
rows?**

Reasons it happens more often than people expect:

- Upstream retraction or a corrected feed
- A filter in the model that can exclude everything for a period
- Genuinely quiet days — a B2B product over a holiday, a region with no activity
- A join that drops all rows when a dimension is late
- Backfilling a range that predates the data

If any of those are possible, you're exposed. If you're unsure, assume you are —
the failure is silent, so absence of evidence is worth nothing here.

## The fixes

**1. Name the partitions explicitly.** The primary fix, and the one that restores
your script's semantics exactly.

```sql
{{ config(
    materialized='incremental',
    incremental_strategy='insert_overwrite',
    partition_by={'field': 'event_date', 'data_type': 'date', 'granularity': 'day'},
    partitions=[
        'date_sub(current_date(), interval 2 day)',
        'date_sub(current_date(), interval 1 day)',
        'current_date()'
    ]
) }}
```

The predicate is now built from your literals, not from the data:

```sql
when not matched by source
     and date(dbt_internal_dest.event_date) in (
         date_sub(current_date(), interval 2 day),
         date_sub(current_date(), interval 1 day),
         current_date()
     )
    then delete
```

That clause fires whether or not the model produced rows. An empty day gets
emptied. **This is the declared range your script had.**

Keep the `partitions` list and your `is_incremental()` filter covering the same
range, or you'll clear partitions you didn't recompute.

**2. Emit a row for every period.** Join to a date spine so the model always
produces at least one row per day, even if the measures are zero. Works, and has
the side benefit of making "zero" explicit downstream rather than absent. Costs
you a spine and some care with joins.

**3. Reconcile on a schedule.** A periodic `--full-refresh` into a scratch
dataset, diffed against production, catches drift from this and every other
silent failure. Not a fix — a detector. Worth having regardless; it's
[J3](../BACKLOG.md#part-j--operating-it-afterwards).

Fix 1 unless you have a specific reason. Fixes 2 and 3 complement it.

## The `NULL` variant

Same mechanism, different trigger. Note `IGNORE NULLS` in the `array_agg` above:
rows whose partition column is `NULL` are never added to the replacement array.

So `NULL`-partition rows **insert** — the insert branch is unconditional — but
never **replace** anything. They accumulate on every single run.

If your partition column is nullable, filter those rows out in the model, or
`coalesce` them to a sentinel date. Your script's `DELETE ... WHERE event_date >=
start_date` also skipped `NULL`s, so this may be pre-existing rather than
introduced — but it's worth checking now while you're looking.

## Test for it before you cut over

Don't reason about this. Run it.

1. Build the model normally so a partition has rows.
2. Add a temporary filter guaranteeing zero rows for that partition —
   `and 1 = 0`, or exclude the date.
3. Run again.
4. Query that partition.

```sql
select count(*) from analytics.daily_events where event_date = '2026-08-31';
```

**Rows still there** ⇒ dynamic path, you're exposed, apply fix 1.
**Zero rows** ⇒ your `partitions` config is doing its job.

Do this on the real model before retiring the script. It takes five minutes and
it is the only check that distinguishes the two behaviours.

## It isn't a bug in dbt

`insert_overwrite` means *overwrite the partitions I produced*. It has no concept
of the partitions you meant to empty, and it can't — nothing in your model
expresses that.

The mismatch is that people read it as *make the target match my model*, which is
what `materialized='table'` does. Once you've seen the `array_agg`, the behaviour
is the only one it could have.

Your script was more explicit than your model is. That's the trade, and naming
the partitions is how you take it back.

---

Previous: [B13 · `DELETE` + `INSERT` → `insert_overwrite`](B13-delete-insert-to-insert-overwrite.md) ·
Next in wave 1: [F17 · When a hook is the wrong answer](../F-hooks/F17-when-a-hook-is-wrong.md) ·
Deeper: [balanced](../../balanced/04-insert-overwrite.md#the-empty-partition-trap) ·
[expert](../../expert/02-generated-sql.md#insert_overwrite--dynamic)
