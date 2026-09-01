# B15 · `TRUNCATE` + `INSERT` → `table`, or `insert_overwrite`

> **Part B — Write-pattern archetypes** · Sourcing: `SRC`
> **The question:** my script empties the whole table and reloads it. Which is that?

Depends on one thing: whether the reload covers everything, or only recent data.

## The unbounded case → `table`

```sql
TRUNCATE TABLE analytics.customer_summary;

INSERT INTO analytics.customer_summary
SELECT customer_id, COUNT(*), SUM(amount) FROM raw.orders GROUP BY 1;
```

Empty everything, rebuild everything. That is `materialized='table'`, and the
conversion is [B1](B1-create-or-replace-to-table.md).

Don't reach for incremental here. The script rebuilds from scratch every run, so
it has no incremental semantics to preserve — you'd be adding behaviour, not
converting it.

## The bounded case → look closer

```sql
TRUNCATE TABLE staging.recent_events;

INSERT INTO staging.recent_events
SELECT * FROM raw.events
WHERE event_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY);
```

This truncates everything but only reloads seven days, so the table *only ever
holds* seven days. That's still a full rebuild of the table's actual contents —
`materialized='table'` with the filter in the model body, no `is_incremental()`
needed:

```sql
{{ config(materialized='table') }}

select * from {{ source('raw', 'events') }}
where event_date >= date_sub(current_date(), interval 7 day)
```

Simpler than the script, and correct.

## When it's really delete-insert

If the `TRUNCATE` is actually a partition-scoped delete written the blunt way —
the table holds history, and the script truncates because it doesn't know how to
delete a range — you have
[B13](B13-delete-insert-to-insert-overwrite.md), not this.

The tell: the `INSERT` covers less than the table is supposed to contain, and
somebody would be upset if history vanished. Check with
[A9](../A-assess/A9-correctness-baseline.md) — does the live table hold more than
the last run inserted?

```sql
select min(event_date), max(event_date), count(distinct event_date)
from analytics.the_table;
```

More distinct dates than the insert range covers ⇒ the truncate isn't wiping
history, so something else is populating it, and you have a second writer
([A5](../A-assess/A5-hidden-state.md)).

## `TRUNCATE` isn't atomic with the insert

Your script has a window where the table is empty. If it fails between the two
statements, the table stays empty.

Both dbt alternatives remove that window — `table` builds and replaces,
`insert_overwrite` deletes and inserts in one `MERGE`. This is a genuine
improvement, and worth noting if anyone has ever seen the table empty at 3am.

---

Previous: [B13 · `DELETE` + `INSERT` → `insert_overwrite`](B13-delete-insert-to-insert-overwrite.md) ·
Next: [B16 · Deduplication scripts](B16-deduplication.md)
