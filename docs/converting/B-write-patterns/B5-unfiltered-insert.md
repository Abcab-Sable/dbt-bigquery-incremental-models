# B5 · Unfiltered `INSERT INTO ... SELECT` → incremental append

> **Part B — Write-pattern archetypes** · Sourcing: `SRC`
> **The question:** my script inserts without any filter. How is that not duplicating?

It probably is. This archetype is the one where converting reveals a pre-existing
bug more often than any other.

## The pattern

```sql
INSERT INTO analytics.event_log (event_id, user_id, occurred_at)
SELECT event_id, user_id, occurred_at
FROM raw.events;
```

No `WHERE`. Every run inserts everything.

## First: find out why it works

There are only a few possibilities, and you need to know which:

**The source is itself transient.** `raw.events` is truncated and reloaded
between runs, so "everything" means "everything new". Common with landing tables.
The dependency on that truncation is invisible and fragile — record it in
[A5](../A-assess/A5-hidden-state.md).

**Something dedups downstream.** The duplicates exist and get filtered later.
That's a compensating hack — [A6](../A-assess/A6-compensating-hacks.md).

**It runs once.** Not a recurring job at all — [A7](../A-assess/A7-what-not-to-convert.md).

**It doesn't work.** The table has duplicates nobody has noticed. Check:

```sql
select event_id, count(*) as n
from analytics.event_log
group by event_id
having count(*) > 1
order by n desc
limit 20;
```

Run this before converting. If it returns rows, the conversion isn't the fix —
the finding is, and it belongs in your baseline so the diff makes sense later.

## The conversion

If the source really is transient and append is genuinely correct:

```sql
{{ config(materialized='incremental') }}

select event_id, user_id, occurred_at
from {{ source('raw', 'events') }}
```

With no `unique_key`, dbt emits `on FALSE` and inserts everything — the same
append semantics your script had. See
[the balanced track](../../balanced/02-choosing-a-strategy.md#merge-without-a-unique_key-is-append-only).

**This preserves the behaviour, including its fragility.** dbt will run this
model in CI, on retry, and during backfills — more often than your script ran.
Every extra run is another full insert.

## The safer conversion

Add a `unique_key` even though the script didn't have one:

```sql
{{ config(
    materialized='incremental',
    unique_key='event_id'
) }}
```

Now a re-run updates rows to identical values instead of duplicating them. The
output is the same when everything goes right, and correct when it doesn't.

Costs more — the merge has to match against the target — but removes an entire
failure class. Unless the volume makes it genuinely infeasible, do this.

If there's no natural key, hash the row:

```sql
select
    to_hex(md5(to_json_string(t))) as _row_key,
    t.*
from {{ source('raw', 'events') }} as t
```

Note this makes identical duplicate rows in the *source* collapse into one, which
may or may not be what you want. Decide deliberately.

## Add a filter if you can

Better still, give it a watermark so it stops reading everything:

```sql
{% if is_incremental() %}
  where occurred_at > (select max(occurred_at) from {{ this }})
{% endif %}
```

That's [B6](B6-watermark-filter.md), and it turns a full-source scan every run
into an incremental read. If the source is large this is where the cost saving
actually comes from.

## Prove it before cutover

```bash
dbt run --select event_log
dbt run --select event_log
```

Counts must match — [E8](../E-translation/E8-idempotency-proving.md). For this
archetype specifically, that two-run check is the whole point.

---

Previous: [B4 · Materialized views](B4-materialized-views.md) ·
Next: [B6 · `INSERT` with a watermark filter](B6-watermark-filter.md)
