# B6 · `INSERT` with a watermark filter → `is_incremental()`

> **Part B — Write-pattern archetypes** · Sourcing: `SRC`
> **The question:** my script reads "everything since the last run". How does dbt express that?

The most common incremental shape there is, and the conversion is nearly
mechanical. The care goes into the boundary.

## The pattern

```sql
INSERT INTO analytics.orders (order_id, customer_id, status, updated_at)
SELECT order_id, customer_id, status, updated_at
FROM raw.orders
WHERE updated_at > (SELECT MAX(updated_at) FROM analytics.orders);
```

## The conversion

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

Three changes. The self-reference becomes `{{ this }}`. The filter is wrapped in
`is_incremental()` so the first build isn't filtered against a table that doesn't
exist. And `unique_key` is added — see below.

`{{ this }}` records no graph dependency, so the self-reference is not a cycle
([E2](../E-translation/E2-ordering-by-ref.md)).

## Why `unique_key` even though the script had none

The script used **strictly greater than**, so it never re-read a row and never
needed to handle updates. That works right up until it doesn't:

- Two rows share the exact max `updated_at`, and one arrives after the run ⇒
  skipped **permanently**
- A row is corrected upstream with an *older* `updated_at` ⇒ never seen
- The model is re-run after a partial failure ⇒ may re-read the boundary row

With `unique_key`, all three become harmless. Without it, dbt emits `on FALSE`
and any re-read is a duplicate.

## The boundary, properly

Strict `>` loses rows that share the boundary timestamp. The usual fix is a small
overlap:

```sql
{% if is_incremental() %}
  where updated_at >= (
      select timestamp_sub(max(updated_at), interval 1 hour) from {{ this }}
  )
{% endif %}
```

You reprocess an hour every run. **With a `unique_key` that's free** — those rows
update themselves to identical values. Without one it's an hour of duplicates
every run, forever.

Size the window to how late your data actually arrives, and write down why:

```sql
-- 1h overlap: vendor batches can arrive up to ~40min late (see DATA-1918)
```

That comment is the difference between a decision and folklore —
[G8](../BACKLOG.md#part-g--scheduling-parameters-backfills).

## Cost: the filter isn't the whole story

Your filter bounds the **source** read. The `MERGE` still has to find matching
rows in the **target**, and on a large unpartitioned table that's a full scan
every run.

Partition the target and bound the merge too:

```sql
{{ config(
    materialized='incremental',
    unique_key='order_id',
    partition_by={'field': 'order_date', 'data_type': 'date'},
    incremental_predicates=[
        "DBT_INTERNAL_DEST.order_date >= date_sub(current_date(), interval 7 day)"
    ]
) }}
```

Read that predicate as a claim about your data: *orders older than 7 days never
change*. If one does, it won't match and will be **inserted as a duplicate**.
More in [B12](B12-extra-predicates.md).

## The watermark that isn't in the table

If the script reads its high-water mark from a *separate* state table rather than
from the target, that's [B7](B7-external-watermark.md) — a different and slightly
worse problem.

---

Previous: [B5 · Unfiltered `INSERT INTO ... SELECT`](B5-unfiltered-insert.md) ·
Next: [B7 · Watermark in a separate state table](B7-external-watermark.md)
