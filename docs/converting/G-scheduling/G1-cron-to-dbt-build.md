# G1 · From cron entries to `dbt build`

> **Part G — Scheduling, parameters, backfills** · Sourcing: `CRAFT`
> **The question:** twelve cron entries become what?

Usually one invocation. The staggered times existed to enforce ordering, and the
DAG does that now — [E2](../E-translation/E2-ordering-by-ref.md).

## The before

```cron
0  2 * * *  bq query < /opt/sql/load_users.sql
15 2 * * *  bq query < /opt/sql/load_orders.sql
30 2 * * *  bq query < /opt/sql/build_summary.sql
45 2 * * *  bq query < /opt/sql/publish.sql
```

Fifteen-minute gaps, chosen because someone measured the first job once and added
margin. If `load_orders` ever takes 20 minutes, `build_summary` reads stale data
and nothing reports it.

## The after

```cron
0 2 * * *  cd /opt/dbt && dbt build --select +publish
```

`+publish` means publish and everything upstream. Ordering derived, parallelism
free, and a slow upstream job delays its dependants rather than being silently
overtaken.

## Use `build`, not `run`

`dbt build` runs models **and their tests** in dependency order, so a failing test
blocks downstream models. `dbt run` then `dbt test` builds everything first, and by
the time a test fails the bad data has already propagated —
[D12](../D-data-movement/D12-assert-gates.md).

For converted models specifically, `build` is the right verb. Put it in the
runbook, because `run` is the more familiar command and people default to it.

## Selecting what to run

| Selector | Runs |
| --- | --- |
| `dbt build` | everything |
| `--select publish` | just that model |
| `--select +publish` | publish and its ancestors |
| `--select publish+` | publish and its descendants |
| `--select tag:daily` | everything tagged `daily` |

Tags are the usual answer for cadence. Tag models by schedule, then one cron entry
per tag — [G2](G2-consolidating-schedules.md).

## The timing thing to get right

Your cron ran at 02:00 local. dbt will too, but the **partition** a model writes
depends on how the model computes "today", and that's UTC unless you say
otherwise.

A job at 02:00 in a UTC+1 summer is 01:00 UTC — same day. At 23:30 local it isn't.
Capture the schedule in UTC in your [A9](../A-assess/A9-correctness-baseline.md)
baseline and check the boundary explicitly —
[H10](../H-verification/H10-reconciling-timestamps.md).

## Don't run both during cutover

The most common cutover incident: cron still firing the old script while dbt runs
the new model, both writing the same table.

Disable the old entry in the same change that enables the new one, and verify:

```sql
select user_email, statement_type, max(creation_time) as last_run
from `region-eu`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
where creation_time > timestamp_sub(current_timestamp(), interval 2 day)
  and destination_table.table_id = 'daily_events'
group by 1, 2;
```

Only the dbt service account should appear —
[H13](../H-verification/H13-sign-off.md).

## What stays outside

Scheduling, retries of the whole invocation, notifications, and anything
event-driven stay in cron or your orchestrator —
[C14](../C-structural/C14-orchestration.md). dbt has no trigger model, and
simulating one is [K4](../K-antipatterns/K4-run-operation-as-scheduler.md).

---

Next: [G2 · Consolidating schedules](G2-consolidating-schedules.md) ·
Back to [the backlog](../BACKLOG.md)
