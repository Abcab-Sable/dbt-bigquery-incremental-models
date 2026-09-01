# A1 · Inventory your scripts

> **Part A — Assess before you convert** · Sourcing: `CRAFT`
> **The question:** what have we actually got?

You cannot sequence a migration you can't see. This is an afternoon's work and it
determines everything after it.

## What to capture

One row per script. Anything more and you won't finish it.

| Field | Why |
| --- | --- |
| Path / identifier | The thing itself |
| Where it runs | Scheduled query, Composer, cron box, someone's laptop |
| Schedule | Feeds [A4](A4-classify-by-trigger.md) and [G1](../G-scheduling/G1-cron-to-dbt-build.md) |
| Target table(s) | The output that matters |
| Owner | The person who can say what "correct" means |
| Last modified | A proxy for whether anyone still understands it |

## Finding them

They are never all in one place. Check, in rough order of yield:

**BigQuery scheduled queries** — the biggest hiding place:

```bash
bq ls --transfer_config --transfer_location=EU --format=prettyjson
```

**Tables with no known producer.** Work backwards from the warehouse — anything
written recently that no inventory row explains has a script behind it:

```sql
select table_schema, table_name, last_modified_time
from `project.region-eu.INFORMATION_SCHEMA.TABLE_STORAGE`
where last_modified_time > timestamp_sub(current_timestamp(), interval 7 day)
order by last_modified_time desc;
```

**Job history by principal.** Service accounts running recurring queries are
scripts whether or not anyone calls them that:

```sql
select user_email, count(*) as jobs, min(creation_time), max(creation_time)
from `region-eu`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
where creation_time > timestamp_sub(current_timestamp(), interval 30 day)
  and statement_type in ('INSERT','MERGE','DELETE','CREATE_TABLE_AS_SELECT','SCRIPT')
group by user_email
order by jobs desc;
```

**Repos, Airflow DAGs, and the ops wiki.** Then ask the team what's missing —
there's always at least one thing running from a machine nobody mentions.

## The rows that matter most

Two categories deserve flagging as you go:

**No owner.** You can't establish a correctness baseline
([A9](A9-correctness-baseline.md)) without someone who knows what right looks
like. Finding the owner is a prerequisite, not a nicety.

**Not modified in over a year, still running daily.** Either it's stable and
boring, or it's been quietly wrong and nobody checks. Both are worth knowing
before you convert.

## What not to do yet

Don't classify, estimate, or prioritise. That's [A3](A3-classify-by-write-pattern.md)
and [A8](A8-estimate-risk.md). An inventory that stalls because someone started
reading the SQL is a common and avoidable failure.

Get the list complete first. It will be longer than anyone expects, and that
number is itself the most useful output of this step.

---

Next: [A2 · Map dependencies](A2-map-dependencies.md) ·
Back to [the backlog](../BACKLOG.md)
