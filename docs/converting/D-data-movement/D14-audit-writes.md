# D14 · Audit and metadata table writes

> **Part D — Data movement, DDL, metadata** · Sourcing: `CRAFT`
> **The question:** my script writes run metadata to a table. Keep it?

Usually replace it. dbt produces richer metadata than most hand-rolled audit
tables, and the hook version has costs that scale badly —
[F13](../F-hooks/F13-post-hook-audit-rows.md) has the mechanics. This page is
about migrating off it without breaking consumers.

## What's usually in these tables

```sql
INSERT INTO ops.job_audit (job_name, run_at, rows_written, duration_s, status)
SELECT 'daily_events', CURRENT_TIMESTAMP(), COUNT(*), @@duration, 'ok'
FROM analytics.daily_events;
```

Job name, timestamp, row count, duration, status. Every one of those is already
in dbt's artefacts or BigQuery's job history.

## The replacements

| Column | Comes from |
| --- | --- |
| job / model name | `run_results.json`, `results[].node.unique_id` |
| run timestamp | `run_results.json`, or `invocation_id` |
| rows written | `results[].adapter_response.rows_affected`, or `dml_statistics` |
| duration | `results[].execution_time` |
| status | `results[].status` |
| bytes billed | `INFORMATION_SCHEMA.JOBS_BY_PROJECT` |

None of it needs a query against your own table, which is the expensive part of
the script's version.

## Load the artefacts

The cleanest replacement, and it covers every model at once:

```bash
dbt build
python scripts/load_run_results.py target/run_results.json
```

One load per invocation, no per-model statements, richer data than the original.

Or from BigQuery's own record, with no extra tooling:

```sql
select
    creation_time,
    destination_table.table_id,
    total_bytes_processed,
    dml_statistics.inserted_row_count,
    dml_statistics.updated_row_count,
    dml_statistics.deleted_row_count,
    timestamp_diff(end_time, start_time, second) as duration_s
from `region-eu`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
where creation_time > timestamp_sub(current_timestamp(), interval 1 day)
  and destination_table.dataset_id = 'analytics';
```

That's your audit table, already populated, retained for 180 days, at no cost.

## Migrating without breaking consumers

The audit table usually has readers — a dashboard, an SLA report, someone's
monitoring. Don't delete it and find out who.

Find them first:

```sql
select user_email, count(*) as n
from `region-eu`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
where creation_time > timestamp_sub(current_timestamp(), interval 90 day)
  and query like '%job_audit%'
group by 1 order by n desc;
```

Then strangle it ([I2](../BACKLOG.md#part-i--migration-strategy)):

1. Keep the existing write, ported as-is to `on-run-end`
2. Start loading `run_results.json` into a new table alongside
3. Move consumers across one at a time
4. Delete the old write and the old table

Steps 1 and 2 running together is the point — nothing breaks while consumers
migrate at their own pace.

## If you must keep writing it

Port to `on-run-end`, not a per-model post-hook — one statement per run instead of
one per model, and it covers failures too
([F14](../F-hooks/F14-on-run-start-end.md)):

```sql
{% macro log_run(results) %}
  {% if execute and results %}
    insert into ops.job_audit (invocation_id, node, status, rows_written, run_at)
    values
    {%- for r in results %}
      ('{{ invocation_id }}', '{{ r.node.unique_id }}', '{{ r.status }}',
       {{ r.adapter_response.get('rows_affected', 'null') }}, current_timestamp())
      {{- ',' if not loop.last }}
    {%- endfor %}
  {% endif %}
{% endmacro %}
```

Note `.get()` with a default — failed nodes have no `rows_affected`, and this must
tolerate them.

## Partition and expire it

Whatever you keep, an audit table appending on every run needs a retention policy
or it becomes its own problem:

```sql
{{ config(
    partition_by={'field': 'run_at', 'data_type': 'timestamp', 'granularity': 'day'},
    partition_expiration_days=90
) }}
```

Scripts rarely did this, which is why these tables are often the oldest and
largest thing in the ops dataset.

---

Previous: [D13 · Notification side-effects](D13-notifications.md) ·
Next: [E9 · Session settings](../E-translation/E9-session-settings.md)
