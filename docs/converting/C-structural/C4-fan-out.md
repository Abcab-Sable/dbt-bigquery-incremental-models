# C4 · One script writing to several tables

> **Part C — Structural archetypes** · Sourcing: `CRAFT`
> **The question:** my script populates four tables. How many models is that?

Four. One model produces one relation, so a script with four outputs becomes four
models — and that's usually an improvement, because the four were never really one
job.

## The pattern

```sql
CREATE OR REPLACE TABLE analytics.daily_users AS SELECT ... ;
CREATE OR REPLACE TABLE analytics.daily_orders AS SELECT ... ;
CREATE OR REPLACE TABLE analytics.daily_revenue AS SELECT ... FROM analytics.daily_orders ... ;
INSERT INTO ops.run_log VALUES ('daily_batch', CURRENT_TIMESTAMP());
```

Four writes. They're in one file because someone scheduled them together, not
because they're one thing.

## The split

Three models plus a decision about the fourth write:

```
models/marts/daily_users.sql      ← independent
models/marts/daily_orders.sql     ← independent
models/marts/daily_revenue.sql    ← ref('daily_orders')
```

The `run_log` insert isn't a model — it's a side effect, and it becomes
`on-run-end` ([F14](../F-hooks/F14-on-run-start-end.md)), not a post-hook on an
arbitrary one of the three.

## What you gain

The script's internal ordering was implicit — `daily_revenue` came third because
it reads `daily_orders`, and that only worked because of line order.

After conversion that's a `ref()`, so:

- `daily_users` and `daily_orders` can run **in parallel**
- adding a dependency later doesn't require remembering to reorder
- rebuilding just `daily_revenue` is `dbt build --select daily_revenue`
- each output can have its own materialization, schedule, and tests

The script gave you none of that, and the parallelism alone often makes the
converted version faster than the original.

## Shared intermediate work

If several outputs derive from a common computation, don't repeat it:

```sql
CREATE TEMP TABLE base AS SELECT ... ;   -- used by all four outputs
```

That's a model of its own — [C3](C3-separate-models.md) — and each output
`ref()`s it. Whether it's `ephemeral`, `view` or `table` depends on cost:
expensive and used four times ⇒ `table`.

Getting this wrong in the other direction is the common mistake: four models each
independently recomputing the same expensive base, because the CTE got copied into
all of them.

## Watch for outputs that aren't tables

Scripts fan out to more than tables:

| Write | Becomes |
| --- | --- |
| Another table | A model |
| An audit/log row | `on-run-end` — [F14](../F-hooks/F14-on-run-start-end.md) |
| A GCS export | Orchestration — [D2](../D-data-movement/D2-export-data.md) |
| A notification | Your scheduler — [D13](../D-data-movement/D13-notifications.md) |
| A view refresh | A model |

Only the first is a model. Sweeping the rest into hooks on whichever model
happens to be last is the failure mode — [F17](../F-hooks/F17-when-a-hook-is-wrong.md).

## The scheduling consequence

The script ran the four together, so they succeeded or failed together. After
conversion, `dbt build` will run what it can — if `daily_users` fails,
`daily_orders` still builds, because nothing connects them.

Usually better. But if the four genuinely must be all-or-nothing, that's a
property you've lost, and you need to decide whether it mattered. Options: a
single `dbt build --select` covering all four with failure handling in your
orchestrator, or accepting partial success as the new behaviour.

Record the decision — it's exactly the kind of thing
[G11](../BACKLOG.md#part-g--scheduling-parameters-backfills) covers, and exactly
the kind nobody notices until an incident.

---

Previous: [C3 · Separate models](C3-separate-models.md) ·
Next: [C5 · `DECLARE` / `SET` scalar variables](C5-declare-set-variables.md)
