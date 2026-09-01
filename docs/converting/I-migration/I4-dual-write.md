# I4 · Dual-write during cutover

> **Part I — Migration strategy** · Sourcing: `CRAFT`
> **The question:** can both write to production while I gain confidence?

No. Both writing to the **same** table is the one thing never to do. What you want
is dual *running* with separate destinations — shadow mode.

## Why same-table dual-write fails

Two producers on one table means:

- Interleaved writes, so the contents depend on execution order
- An `insert_overwrite` from one clearing partitions the other just wrote
- A `merge` from one seeing rows the other inserted
- Row counts that match neither implementation
- No way to attribute a difference to either

You lose the ability to say what either produces, which defeats the entire point
of running both.

## Shadow, then swap

```
analytics.daily_events          ← script keeps writing, untouched
analytics_shadow.daily_events   ← model writes here
```

Compare daily ([H5](../H-verification/H5-shadow-mode.md)). When the criteria in
[H6](../H-verification/H6-shadow-duration.md) are met, make **one** change:

1. Disable the script's schedule
2. Point the model at `analytics`
3. Deploy both together

Between those two states there is never a moment with two writers.

## Verify the old one actually stopped

The most common cutover incident is a schedule that didn't get disabled — a
duplicate cron entry, a second scheduled query, a paused DAG someone resumed.

```sql
select user_email, statement_type, max(creation_time) as last_run, count(*) as n
from `region-eu`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
where creation_time > timestamp_sub(current_timestamp(), interval 3 day)
  and destination_table.table_id = 'daily_events'
group by 1, 2
order by last_run desc;
```

Only the dbt service account should appear after cutover. Run this the day after,
not just on the day.

## The genuine dual-write case

There is one legitimate version: writing to **two different tables**, both
maintained, while consumers migrate.

```
analytics.daily_events        ← script, for old consumers
analytics.daily_events_v2     ← model, for migrated consumers
```

Consumers move at their own pace. Both are correct; neither is a shadow.

The cost is real — two implementations to keep in step, twice the compute, and
someone will change one and not the other. Only do it when consumers genuinely
cannot move together, and set a deadline for retiring the old table.

## Don't dual-write from one model

Tempting:

```sql
{{ config(post_hook="insert into analytics.daily_events select * from {{ this }}") }}
```

A model writing to a second table via a hook. This is
[F17](../F-hooks/F17-when-a-hook-is-wrong.md) — the second table has no lineage,
no tests, and no independent build. And the hook doesn't run if the model fails,
so the two silently diverge on the first bad day.

If two tables are needed, that's two models.

## Have a rollback ready

Before cutting over, be able to answer: **if this is wrong tomorrow, what do we
do?**

- The script is disabled but intact, re-enablable in minutes —
  [I6](I6-rollback-keeping-script.md)
- A baseline snapshot exists from [A9](../A-assess/A9-correctness-baseline.md)
- Someone is around to notice, and knows what to look for

Cutting over on a Friday afternoon with the author on leave is where this goes
wrong.

---

Previous: [I3 · Converting while others depend on the output](I3-converting-with-dependents.md) ·
Next: [I5 · Telling downstream consumers](I5-notifying-consumers.md)
