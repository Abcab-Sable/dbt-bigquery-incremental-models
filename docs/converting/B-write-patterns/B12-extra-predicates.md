# B12 · Extra `ON` predicates → `incremental_predicates`

> **Part B — Write-pattern archetypes** · Sourcing: `SRC`
> **The question:** my `ON` clause has more than key equality. Where does the rest go?

Into `incremental_predicates`. The important part is understanding that these are
claims about your data, not just performance settings.

## Splitting the `ON` clause

```sql
ON  T.order_id = S.order_id                                    -- key
AND T.tenant_id = S.tenant_id                                  -- key
AND T.order_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)   -- a bound
AND T.is_archived = FALSE                                      -- a filter
```

Key equality ⇒ `unique_key`. Everything else ⇒ `incremental_predicates`:

```sql
{{ config(
    materialized='incremental',
    unique_key=['order_id', 'tenant_id'],
    incremental_predicates=[
        "DBT_INTERNAL_DEST.order_date >= date_sub(current_date(), interval 7 day)",
        "DBT_INTERNAL_DEST.is_archived = false"
    ]
) }}
```

Use dbt's aliases — `DBT_INTERNAL_DEST` for the target, `DBT_INTERNAL_SOURCE` for
the incoming rows. Your original `T` and `S` don't exist in the generated SQL.

Predicates are joined with `AND`, exactly as in your `ON` clause.

## Both config names work

Read as:

```jinja
config.get('predicates', default=none) or config.get('incremental_predicates', default=none)
```

`predicates` wins if both are set. Pick one and be consistent; the long name is
clearer.

## The part that isn't performance

A predicate on the **target** side restricts what can match. Rows outside it
cannot match, so they are **inserted rather than updated**.

`DBT_INTERNAL_DEST.order_date >= date_sub(current_date(), interval 7 day)` says:
*orders older than seven days never change.* If a 30-day-old order does change,
it won't find its existing row, and you get a duplicate.

That's a statement about your business, and it needs to be true. Write down why:

```sql
incremental_predicates=[
    -- Orders finalise at T+5d; nothing older is ever amended (see DATA-2104).
    "DBT_INTERNAL_DEST.order_date >= date_sub(current_date(), interval 7 day)"
]
```

Without the comment it looks like a tuning knob, and someone will widen or narrow
it without realising they've changed correctness.

## Make it prunable

The point of the bound is to let BigQuery skip partitions. That only works if the
partition column appears **bare** on one side:

```sql
-- prunes
"DBT_INTERNAL_DEST.order_date >= date_sub(current_date(), interval 7 day)"

-- does not prune
"cast(DBT_INTERNAL_DEST.order_date as string) >= '2026-08-25'"
"date_trunc(DBT_INTERNAL_DEST.order_date, month) >= '2026-08-01'"
```

Both give correct results. The second reads the whole table. There's no warning —
just the bill.

## `require_partition_filter` won't do this for you

If the target has `require_partition_filter`, dbt injects a predicate
automatically:

```sql
(DBT_INTERNAL_DEST.order_date is null or DBT_INTERNAL_DEST.order_date is not null)
```

A tautology. It satisfies BigQuery's requirement and restricts nothing, so you
get no pruning from it. You still need a real bound. See
[the balanced track](../../balanced/03-merge.md#the-require_partition_filter-predicate).

## Check what your script's predicate meant

Before porting a bound, work out whether it was for cost or for correctness:

- **Cost** ⇒ port it, tune it freely, measure the effect
- **Correctness** (a tenant filter, an archived-row guard) ⇒ port it exactly, and
  treat changes as behaviour changes

Getting these confused in either direction causes problems. A correctness filter
loosened "for performance" changes results; a cost bound treated as sacred stops
anyone tuning it.

---

Previous: [B11 · Conditional `WHEN MATCHED AND ...`](B11-conditional-when-matched.md) ·
Next: [B15 · `TRUNCATE` + `INSERT`](B15-truncate-insert.md)
