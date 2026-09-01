# B7 · Watermark held in a separate state table

> **Part B — Write-pattern archetypes** · Sourcing: `CRAFT`
> **The question:** my script reads and writes a `last_run` table. What replaces it?

`{{ this }}`. The state table exists because a script has nowhere else to keep
state — an incremental model does.

## The pattern

```sql
DECLARE last_run TIMESTAMP;
SET last_run = (SELECT watermark FROM ops.job_state WHERE job = 'orders');

INSERT INTO analytics.orders
SELECT * FROM raw.orders WHERE updated_at > last_run;

UPDATE ops.job_state
SET watermark = (SELECT MAX(updated_at) FROM analytics.orders)
WHERE job = 'orders';
```

Three statements, two tables, one purpose.

## Why it exists, and why it can go

The state table answers "where did I get to". So does the target table:

```sql
select max(updated_at) from {{ this }}
```

The target already knows, because it contains the rows. Keeping the answer
somewhere else means keeping two things in sync, and the failure mode is that
they drift.

```sql
{{ config(
    materialized='incremental',
    unique_key='order_id'
) }}

select order_id, customer_id, status, updated_at
from {{ source('raw', 'orders') }}

{% if is_incremental() %}
  where updated_at > (select max(updated_at) from {{ this }})
{% endif %}
```

The `DECLARE`, the `SET`, and the `UPDATE` all disappear. Only the middle
statement was ever the model — [E1](../E-translation/E1-one-statement-per-model.md).

## The failure the state table has and `{{ this }}` doesn't

The script's watermark update is a **separate statement** from the insert. If the
insert succeeds and the update fails, the watermark is stale and the next run
reprocesses. If the update somehow lands without the insert, rows are skipped
permanently.

Reading `max()` from the target can't desynchronise, because there's nothing to
synchronise. The watermark is derived, not stored.

Before converting, check whether they're currently in sync — a mismatch is a
finding for [A9](../A-assess/A9-correctness-baseline.md):

```sql
select
    (select watermark from ops.job_state where job = 'orders') as stored,
    (select max(updated_at) from analytics.orders)             as actual;
```

If `stored` is behind `actual`, some rows have been reprocessed repeatedly. If
it's ahead, rows were skipped, and there's a gap in your data that predates the
conversion.

## When the state table has other readers

Check before deleting it:

```sql
select user_email, statement_type, count(*) as n
from `region-eu`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
where creation_time > timestamp_sub(current_timestamp(), interval 90 day)
  and query like '%job_state%'
group by 1, 2;
```

If monitoring, alerting, or another job reads it, the table stays until those
move. Keep it updated from a post-hook **as a temporary measure**, with a ticket
and a comment — see [F17](../F-hooks/F17-when-a-hook-is-wrong.md) on why that's a
stopgap and not a destination.

## The case where it's genuinely needed

If the watermark column isn't in the target — the model reads `updated_at` but
only stores `order_date`, say — there's nothing to take `max()` of.

Fix it by carrying the column into the model. An incremental model that can't
determine its own position from its own contents is fighting the design, and
adding one column is cheaper than keeping a second table in sync forever.

---

Previous: [B6 · `INSERT` with a watermark filter](B6-watermark-filter.md) ·
Next: [B9 · `WHEN MATCHED THEN UPDATE` → update columns](B9-when-matched-update.md)
