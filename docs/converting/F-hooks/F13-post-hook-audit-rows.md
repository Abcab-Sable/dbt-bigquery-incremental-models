# F13 · post-hook: writing audit rows

> **Part F — Hooks** · Sourcing: `CRAFT`
> **The question:** my script logs each run to an audit table. Keep it?

Probably not in that form. dbt already produces richer run metadata than most
hand-rolled audit tables, and a per-model hook that writes rows has costs that
scale badly.

## The pattern

```sql
INSERT INTO ops.job_audit (job_name, run_at, rows_written)
SELECT 'daily_events', CURRENT_TIMESTAMP(), COUNT(*) FROM analytics.daily_events;
```

Ported straight across:

```sql
{{ config(post_hook="""
    insert into ops.job_audit (job_name, run_at, rows_written)
    select '{{ this.identifier }}', current_timestamp(), count(*) from {{ this }}
""") }}
```

Works. Three problems.

## The problems

**It scans the table every run.** `count(*)` on a partitioned table is cheap-ish
but not free, and on a large unpartitioned one it isn't cheap at all. You've added
a full scan to every build, attributed to the model's runtime.

**It doesn't scale across models.** Fine for one. At 300 models it's 300 extra
statements and 300 scans per run, and the audit table grows by 300 rows a day for
information dbt already has.

**It isn't atomic with the model.** The hook is a separate statement. A failure
between the model and the hook leaves the model built and unaudited; there's no
rollback — [F16](F16-hooks-and-failure.md).

## What dbt already gives you

**`run_results.json`**, written every invocation, containing per-node status,
timing, and adapter response including `rows_affected` and `bytes_processed`. No
extra query, no extra statement.

Load it into the warehouse once per run and you have a better audit table than the
hook produced, at a fraction of the cost:

```bash
dbt run
python scripts/load_run_results.py target/run_results.json
```

**`on-run-end`**, which fires once per invocation with access to the results
context — one statement per run rather than per model.
See [F14](F14-on-run-start-end.md).

**BigQuery's own job history**, which records every statement dbt issued:

```sql
select
    creation_time,
    destination_table.table_id,
    total_bytes_processed,
    dml_statistics.inserted_row_count,
    dml_statistics.updated_row_count,
    dml_statistics.deleted_row_count
from `region-eu`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
where creation_time > timestamp_sub(current_timestamp(), interval 1 day)
  and destination_table.dataset_id = 'analytics';
```

That gives you rows written, bytes billed, and duration — without adding a single
statement to your project.

## If you keep the hook

Sometimes the audit table has external consumers you can't migrate quickly. If so:

**Don't `count(*)`.** Use the adapter response if you can, or accept logging
without a row count. The count is usually the expensive part and the least used
column.

**Put it at project level**, not per model, so it's defined once:

```yaml
models:
  my_project:
    +post_hook: "{{ audit_model_run(this) }}"
```

**Make the target partitioned**, or the audit table becomes its own problem within
a year.

**Set an expiry.** Audit rows nobody reads after 90 days shouldn't be stored for
five years.

**Write down why it exists** and who reads it, with a ticket to retire it —
[F17](F17-when-a-hook-is-wrong.md).

## The migration path

1. Keep the hook, so nothing breaks.
2. Start loading `run_results.json` alongside it.
3. Point consumers at the new table.
4. Delete the hook.

That's the strangler pattern applied to a hook —
[I2](../I-migration/I2-strangler-pattern.md) — and it's the right shape here
because the audit table's readers are usually dashboards nobody wants to break
during a conversion.

---

Previous: [F12 · post-hook: table options](F12-post-hook-table-options.md) ·
Next: [F14 · `on-run-start` / `on-run-end`](F14-on-run-start-end.md)
