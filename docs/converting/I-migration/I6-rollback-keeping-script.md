# I6 · Rollback: keeping the old script runnable

> **Part I — Migration strategy** · Sourcing: `CRAFT`
> **The question:** if the converted model is wrong, how do we get back?

Re-enable the script. Which requires that you disabled it rather than deleted it,
and that it still works — neither of which is automatic.

## Disable, don't delete

At cutover:

- **Do** turn off the schedule
- **Do** leave the script file in place, in version control
- **Do** leave any supporting objects it needs
- **Don't** delete the script, its temp datasets, or its service account
- **Don't** drop the tables it read

Deletion is [I8](I8-decommissioning.md), and it comes after the rollback window,
not with the cutover.

## Write the rollback down

An untested rollback plan is a hope. Make it a procedure someone else could
execute at 3am:

```
ROLLBACK — analytics.daily_events

1. Disable the dbt schedule:
   comment out `tag:daily` entry in deploy/cron.d/dbt

2. Re-enable the script:
   bq mk --transfer_config --params=... (saved in ops/restore_daily_events.sh)

3. Restore the table if the model corrupted it:
   create or replace table analytics.daily_events as
   select * from analytics_baseline.daily_events_20260901;
   -- then run the script once to catch up

4. Notify #data-platform and the consumers listed in exposures.

Owner: Alice. Tested: 2026-09-01 in scratch dataset.
Window closes: 2026-09-30 (after month-end close).
```

Step 3 is why [A9](../A-assess/A9-correctness-baseline.md) insists on a real
snapshot table. BigQuery time travel is seven days, which is usually shorter than
the window you want — and it doesn't survive a drop-and-recreate
([E13](../E-translation/E13-time-travel.md)).

## Test it before you need it

Run the rollback in a scratch dataset before cutover. You'll discover at least one
of:

- The script references a temp dataset someone cleaned up
- Its service account lost a permission during the migration
- It reads a table your conversion renamed
- Nobody has the credentials to re-enable the scheduled query

All cheap to fix in advance. All painful to discover during an incident.

## How long to keep it

Long enough to cover a full reporting cycle, and specifically:

- A **month-end close**, if finance reads it
- A **quarter end**, if anyone reports quarterly
- Whatever period the [H6](../H-verification/H6-shadow-duration.md) shadow run
  didn't cover

Two weeks is a common default and is often too short — monthly consumers won't
have looked yet.

## The catch-up problem

Rolling back after a week means the script must process a week of data it missed.
Whether it can depends on its shape:

- **`insert_overwrite` with a lookback** ⇒ widen the window and run
- **Watermark-based append** ⇒ probably fine, it reads from `max()`
- **Fixed "yesterday" logic** ⇒ needs manual dates, once per missed day

Note which case yours is in the rollback plan. A script that can only ever do
yesterday needs a loop, and discovering that mid-incident is the worst time.

## When rollback stops being viable

At some point the model's output has been consumed, downstream tables built from
it, and reverting would create more inconsistency than it fixes. That's
[I7](I7-rollback-non-viable.md), and it's worth knowing when you cross it —
because after that point the only way is forward.

---

Previous: [I5 · Telling downstream consumers](I5-notifying-consumers.md) ·
Next: [I7 · When rollback stops being viable](I7-rollback-non-viable.md)
