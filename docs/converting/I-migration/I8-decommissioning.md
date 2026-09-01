# I8 · Decommissioning checklist

> **Part I — Migration strategy** · Sourcing: `CRAFT`
> **The question:** the window has closed. What do I actually delete?

More than the script, and in an order that leaves you able to stop halfway.

## The checklist

```
DECOMMISSION — daily_events.sql

Preconditions
  [ ] Sign-off complete                          → H13
  [ ] Rollback window closed                     → I7
  [ ] No non-dbt writer in 30 days (query below)

Schedules and code
  [ ] Scheduled query / cron entry deleted (not just paused)
  [ ] Script file removed from the repo
  [ ] Any wrapper (shell, DAG task) removed
  [ ] Dead Airflow DAG or operator deleted

dbt project
  [ ] `legacy` source entry removed                → E3
  [ ] No `source('legacy', 'daily_events')` remains
  [ ] Exposures for converted consumers removed    → I3

Infrastructure
  [ ] Temp datasets the script used dropped
  [ ] Service account retired, if used only by this
  [ ] Alerts/monitors pointing at the script removed
  [ ] Secrets it used rotated or deleted

Data
  [ ] Intermediate tables the script wrote dropped
  [ ] Baseline snapshot retained until ____ (later than the window)

Documentation
  [ ] Runbook updated                              → I10
  [ ] Wiki page pointing at the script updated or deleted
  [ ] Inventory row marked done                    → A1
```

## The check that matters most

Before deleting anything, confirm nothing but dbt has written to the table:

```sql
select
    user_email,
    statement_type,
    count(*) as n,
    max(creation_time) as last_run
from `region-eu`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
where creation_time > timestamp_sub(current_timestamp(), interval 30 day)
  and destination_table.table_id = 'daily_events'
group by 1, 2
order by last_run desc;
```

Only the dbt service account. Anything else is a writer you didn't know about —
stop and find it.

## Delete the schedule, don't pause it

A paused scheduled query is a loaded gun. Someone doing housekeeping in a year
sees it paused, assumes it's dormant rather than dead, and resumes it. Now two
things write the table and nobody connects the two events.

Delete it, and keep the definition in version control if you want the record.

## Drop the intermediate tables

Scripts leave debris — temp datasets, `_staging` tables, `_old` copies. They cost
storage and they confuse whoever comes next.

Find them:

```sql
select table_schema, table_name, last_modified_time,
       round(total_logical_bytes / pow(10,9), 2) as gb
from `project.region-eu.INFORMATION_SCHEMA.TABLE_STORAGE`
where last_modified_time < timestamp_sub(current_timestamp(), interval 60 day)
order by total_logical_bytes desc;
```

Anything untouched since the cutover, that the conversion replaced, is a
candidate. Check each rather than bulk-deleting — one of them will turn out to be
read quarterly.

## Keep the baseline snapshot longer

The one thing not to delete on schedule. Keep
[A9](../A-assess/A9-correctness-baseline.md)'s snapshot well past the rollback
window — it's your only evidence of what the old output looked like, and questions
about "did this number change" arrive months later.

Set an expiry rather than keeping it forever:

```sql
alter table analytics_baseline.daily_events_20260901
set options(expiration_timestamp = timestamp '2026-12-01 00:00:00 UTC');
```

## Stop-anywhere ordering

The list is ordered so you can stop after any section without leaving something
dangerous. Schedules first (that's the double-write risk), then code, then
project, then infrastructure, then data.

Never delete data before confirming nothing writes to it.

---

Previous: [I7 · When rollback stops being viable](I7-rollback-non-viable.md) ·
Next: [I9 · What to keep from the old script](I9-what-to-keep.md)
