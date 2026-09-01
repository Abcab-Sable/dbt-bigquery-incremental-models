# I9 · What to keep from the old script

> **Part I — Migration strategy** · Sourcing: `CRAFT`
> **The question:** the script is going. Is any of it worth preserving?

The intent. The code is replaced; the reasoning behind it usually isn't written
down anywhere else, and it's the expensive part to reconstruct.

## Keep the comments that explain why

Scripts accumulate comments recording decisions:

```sql
-- Vendor 447 double-sends on retry (INC-4471). Exclude until they fix it.
-- 7-day window: their batches can land up to 5 days late.
-- DO NOT change to INNER JOIN — we need rows with no matching user.
```

None of that survives in the SQL you write, because the SQL looks different. Carry
each one across to the model, next to the logic it explains:

```sql
select ...
from {{ source('raw', 'orders') }}
-- Vendor 447 double-sends on retry (INC-4471, open with vendor).
-- Remove when their fix ships.
where vendor_id != 447
```

This is the same material as [A6](../A-assess/A6-compensating-hacks.md), arriving
at its destination.

## Keep the git history

Don't `rm` the script in the same commit that adds the model, if you can avoid it.
Two commits — add the model, then remove the script — makes the history readable
and keeps `git log --follow` useful for a while.

If the script lived in a different repo, note where:

```yaml
models:
  - name: daily_events
    description: >
      Daily event counts by user.
      Converted from daily_events.sql (was in acme/etl-scripts,
      last version at commit 3f9a21c). See DATA-2288.
```

The commit hash costs nothing and answers "what did it used to do" in one command.

## Keep the baseline

[A9](../A-assess/A9-correctness-baseline.md)'s document is the record of what the
old thing did and why the model is shaped the way it is. Commit it next to the
model:

```
models/events/daily_events.sql
models/events/daily_events.baseline.md
```

Six months later it answers: what were the row counts, was it idempotent, which
compensating hacks existed, what did we deliberately change.

## Keep the decisions that look arbitrary

Anything in the model that a reader would want to change should say why it's
there:

```sql
{{ config(
    materialized='incremental',
    incremental_strategy='insert_overwrite',
    partition_by={'field': 'event_date', 'data_type': 'date', 'granularity': 'day'},
    -- Static partitions, not dynamic: refunds-only days produce zero rows,
    -- and dynamic insert_overwrite would leave stale data. See B14.
    partitions=[
        'date_sub(current_date(), interval 1 day)',
        'current_date()'
    ]
) }}
```

Without that comment, someone will "simplify" it to the dynamic form and
reintroduce [the trap](../B-write-patterns/B14-when-the-range-can-empty.md). The
comment is the only thing standing between the model and that change.

## What not to keep

- **The SQL itself**, beyond the git history. A commented-out script in the repo
  rots and confuses.
- **Dead compensating hacks** you deliberately dropped — but record *that* you
  dropped them, in the baseline.
- **The old table**, past the retention you decided in
  [I8](I8-decommissioning.md).
- **Wiki pages describing the script's schedule and ordering.** That's the DAG
  now, and a stale wiki is worse than none.

## The test

> **If everyone who worked on this leaves, can the next person understand why the
> model looks like this?**

If the answer depends on someone remembering, write it down before the script goes.
That's the whole of this unit.

---

Previous: [I8 · Decommissioning checklist](I8-decommissioning.md) ·
Next: [I10 · Documenting the conversion](I10-documenting-decisions.md)
