# K2 · Hooks as a general-purpose escape hatch

> **Part K — Anti-patterns** · Sourcing: `CRAFT`
> **The question:** what's wrong with putting the leftover statements in a post-hook?

It turns the model back into a script, with less visibility than the script had.
This is the most common bad outcome of a conversion.

## The shape

```sql
{{ config(
    materialized='table',
    post_hook=[
        "insert into analytics.daily_summary select ... from {{ this }}",
        "delete from analytics.daily_events where event_date < date_sub(current_date(), interval 90 day)",
        "export data options(uri='gs://...') as select * from {{ this }}",
        "insert into ops.audit values ('daily_events', current_timestamp())"
    ]
) }}
```

Four statements attached to a model. The script had four statements. Nothing has
been converted — it's been relocated.

## What's been given up

Each of those hook statements has:

- **No DAG node.** Nothing can `ref()` `daily_summary`; it isn't in the lineage.
- **No tests.** You can't assert anything about a table dbt doesn't know exists.
- **No selection.** `dbt build --select daily_summary` doesn't work.
- **No failure isolation.** Hook 2 failing means 3 and 4 don't run, and 1's effect
  stands. No rollback — [F16](../F-hooks/F16-hooks-and-failure.md).
- **Ordering by accident.** These run because they're attached to *this* model,
  which is an arbitrary choice.

The script had the same problems. But it *looked* like a script, so nobody
expected lineage. A model that looks declarative and isn't is worse — it misleads.

## The tell

**DML in a hook against a table that isn't this model.** `INSERT`, `MERGE`,
`UPDATE`, `DELETE`, `CREATE TABLE AS` pointed anywhere else means you're building
a second model inside the first.

Grep for it:

```bash
grep -rniE "post_hook|pre_hook" models/ | grep -iE "insert|merge|update|delete|create table"
```

Every hit is a candidate.

## Where each one goes

| Hook statement | Should be |
| --- | --- |
| `insert into another_table select ...` | **A model** — [C4](../C-structural/C4-fan-out.md) |
| `delete ... where date < x` (retention) | A scheduled operation, not a model hook |
| `export data ...` | Orchestration — [D2](../D-data-movement/D2-export-data.md) |
| `insert into ops.audit ...` | `on-run-end` — [F14](../F-hooks/F14-on-run-start-end.md) |
| `grant ...` | The `grants` config — [F11](../F-hooks/F11-grants-vs-post-hook.md) |
| A data-quality check | A dbt test — [D12](../D-data-movement/D12-assert-gates.md) |

Notice the pattern: almost every abused hook has a first-class dbt feature that
does the job **with** lineage, retries and visibility.

## Why it happens

The hook is the path of least resistance. You've converted the main query, four
statements are left, and `post_hook` accepts a list of strings without complaint.

It also feels like progress — the script file is gone. But the work it was doing
is now distributed into config strings nobody greps.

## The rule

> **If you can't explain why it isn't a model, it should be a model.**

And the check from [F10](../F-hooks/F10-post-hook-patterns.md): *if this fails,
should the model be considered failed?* Yes ⇒ it's part of the model. No ⇒ it
shouldn't be attached to the model's build.

Only metadata and audit sit legitimately in between.

## Fixing it later

If you inherit this, convert one hook at a time. Extract the DML into a model,
point consumers at it, delete the hook. Same strangler shape as
[I2](../I-migration/I2-strangler-pattern.md) — and each extraction adds a node to
a lineage graph that's currently lying to you.

---

Previous: [K1 · The mega-model](K1-mega-model.md) ·
Next: [K3 · Incremental where a table belongs](K3-unnecessary-incremental.md)
