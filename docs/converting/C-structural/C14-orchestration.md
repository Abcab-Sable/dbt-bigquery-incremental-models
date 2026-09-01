# C14 · Shell, `bq` CLI, and Airflow orchestration

> **Part C — Structural archetypes** · Sourcing: `CRAFT`
> **The question:** our pipeline is a shell script full of `bq query`. What converts and what stays?

The queries convert. The orchestration mostly stays — but it gets much smaller,
because dbt takes over the ordering.

## The pattern

```bash
#!/bin/bash
set -e

bq query --use_legacy_sql=false < sql/01_load_users.sql
bq query --use_legacy_sql=false < sql/02_load_orders.sql
bq query --use_legacy_sql=false < sql/03_build_summary.sql

if [ $? -ne 0 ]; then
    curl -X POST "$SLACK_WEBHOOK" -d '{"text":"daily batch failed"}'
fi
```

Three queries in sequence, plus a notification.

## The split

| Part | Where it goes |
| --- | --- |
| Each `.sql` file | A model |
| The sequence | The DAG — [E2](../E-translation/E2-ordering-by-ref.md) |
| `set -e` / failure propagation | dbt's own behaviour |
| The Slack notification | **Stays** in the orchestrator |
| The schedule | **Stays** — [G1](../BACKLOG.md#part-g--scheduling-parameters-backfills) |

The shell script becomes:

```bash
#!/bin/bash
set -e
dbt build --select +daily_summary
```

Plus whatever notification wrapper you had. The ordering, the `set -e`, and the
per-file invocation all disappear.

## Airflow: the same shape

```python
load_users = BigQueryInsertJobOperator(task_id='load_users', ...)
load_orders = BigQueryInsertJobOperator(task_id='load_orders', ...)
build_summary = BigQueryInsertJobOperator(task_id='build_summary', ...)

load_users >> load_orders >> build_summary
```

The `>>` chain is a DAG expressed twice — once in Airflow, once implicitly in the
SQL. After conversion there's one DAG, and Airflow holds a single task:

```python
dbt_build = BashOperator(
    task_id='dbt_build',
    bash_command='dbt build --select +daily_summary',
)
```

## What legitimately stays in the orchestrator

Don't try to move these into dbt:

- **Scheduling and triggers**, including event-driven ones —
  [A4](../A-assess/A4-classify-by-trigger.md)
- **Notifications** — [D13](../D-data-movement/D13-notifications.md)
- **Retries** of the whole invocation
- **Cross-system dependencies** — waiting for a file, an API, another team's job
- **Secrets and credential management**
- **Non-dbt steps** — exports, ML jobs, third-party syncs

dbt is a transformation tool. It has no trigger model and no notion of external
systems, and simulating either inside it is
[K4](../BACKLOG.md#part-k--anti-patterns).

## Granularity: one task or several?

After converting, you have a choice:

**One task, `dbt build`.** Simplest. dbt handles ordering and parallelism
internally. Failure of one model doesn't stop unrelated branches. Preferred
default.

**Several tasks, using `--select`.** More Airflow-visible granularity, per-group
retries, and the ability to interleave non-dbt steps. Costs you a `dbt parse` per
task and reintroduces some ordering into Airflow.

Start with one. Split only when you have a concrete reason — usually an external
step that must run between two groups of models.

## The failure-semantics change

`set -e` stopped at the first failure. `dbt build` **continues** — it skips the
failed model's descendants and builds everything else.

That's usually better, but it's a change. After a partial failure you have a
partially-updated warehouse rather than a cleanly-stopped one. Know it before it
happens at 3am, and decide whether downstream consumers can tolerate it —
[G11](../BACKLOG.md#part-g--scheduling-parameters-backfills).

If you need all-or-nothing, that's `--fail-fast`, and it's a deliberate choice.

## Don't convert the wrapper first

Tempting to start by replacing the shell script with a `dbt build` call. Don't —
until the models exist, there's nothing to build.

Convert leaf queries to models, let the shell script keep invoking the old
files for the rest, and shrink it as you go. Same strangler approach as
[C12](C12-nested-procedures.md).

---

Previous: [C13 · Python scripts](C13-python-scripts.md) ·
Next: [D1 · `LOAD DATA` from GCS](../D-data-movement/D1-load-data.md)
