# B4 · Materialized view managed by script → the MV materialization

> **Part B — Write-pattern archetypes** · Sourcing: `SRC`
> **The question:** my script manages a BigQuery materialized view. Does dbt?

Yes — dbt-bigquery ships a `materialized_view` materialization. The conversion is
straightforward; the thing to check is whether you wanted an MV in the first
place.

## The conversion

```sql
CREATE MATERIALIZED VIEW analytics.daily_totals
PARTITION BY event_date
AS SELECT event_date, SUM(amount) AS total
FROM analytics.events
GROUP BY event_date;
```

```sql
{{ config(
    materialized='materialized_view',
    partition_by={'field': 'event_date', 'data_type': 'date', 'granularity': 'day'}
) }}

select event_date, sum(amount) as total
from {{ ref('events') }}
group by event_date
```

## What an MV is, and isn't

A BigQuery materialized view is **maintained by BigQuery**, not by your schedule.
It refreshes automatically as the base table changes, and queries against the
base table can be transparently rewritten to use it.

That's a different thing from an incremental model, and the distinction decides
whether the conversion is right:

| | Materialized view | Incremental model |
| --- | --- | --- |
| Refreshed by | BigQuery, automatically | Your dbt run |
| Query flexibility | Restricted — limited SQL surface | Anything |
| Cost model | Storage plus maintenance | Storage plus your run |
| Can it be wrong? | No — BigQuery keeps it consistent | Yes, silently |

That last row is the real argument for MVs where they fit. An MV cannot drift,
because you aren't the one maintaining it.

## The restrictions are the catch

BigQuery MVs support a narrow slice of SQL. No outer joins in many cases, limited
aggregate support, no window functions, restrictions on nested queries. If the
script's MV exists, the query already satisfies them — but any change you make
during conversion might not.

**Convert the query verbatim first.** Tidying it and converting it in the same
change is [K11](../K-antipatterns/K11-convert-and-optimise.md), and here it can turn a
working MV into a compile error for non-obvious reasons.

## When the script is doing BigQuery's job

The pattern worth looking for:

```sql
CREATE OR REPLACE MATERIALIZED VIEW ... ;   -- rebuilt on a schedule
```

If the script *drops and recreates* the MV on a schedule, it isn't using the MV's
automatic maintenance at all — it's using an MV as an oddly expensive table. That
converts to `materialized='table'` or an incremental model, not to
`materialized_view`.

Check the schedule from [A4](../A-assess/A4-classify-by-trigger.md). An MV that
gets recreated nightly is a table with extra steps.

## Changing an MV

MVs generally can't be altered in place — changing the query means dropping and
recreating, which loses the maintenance state and triggers a full recompute.
Expect the first run after a change to be expensive, and don't iterate on an MV
model in production.

## When to move away from it

If the MV's restrictions are forcing awkward SQL — the query is contorted to stay
within what MVs allow — converting to an incremental model may be the better
outcome. You take on the drift risk this documentation is about, and get your
query back.

That's a design decision, not a conversion. Make it separately.

---

Previous: [B3 · `CREATE VIEW` → `view`](B3-create-view.md) ·
Next: [B5 · Unfiltered `INSERT INTO ... SELECT`](B5-unfiltered-insert.md)
