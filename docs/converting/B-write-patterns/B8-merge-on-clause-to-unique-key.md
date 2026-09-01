# B8 · The `MERGE` `ON` clause → `unique_key`

> **Part B — Write-pattern archetypes** · Sourcing: `SRC`
> **The question:** I have a hand-written `MERGE`. Which parts become config?

A hand-written `MERGE` is the friendliest thing to convert, because dbt's `merge`
strategy generates the same statement. You're not changing behaviour — you're
deleting the parts dbt writes for you and keeping the parts it can't know.

This page covers the `ON` clause. The other clauses have their own units:
[B9](../BACKLOG.md#part-b--write-pattern-archetypes) for `WHEN MATCHED`,
[B10](../BACKLOG.md#part-b--write-pattern-archetypes) for
`WHEN NOT MATCHED BY SOURCE`, [B11](../BACKLOG.md#part-b--write-pattern-archetypes)
for conditional clauses, [B12](../BACKLOG.md#part-b--write-pattern-archetypes) for
extra predicates.

## The starting point

```sql
MERGE INTO analytics.orders AS T
USING (
    SELECT order_id, customer_id, status, amount, updated_at
    FROM raw.orders
    WHERE updated_at > (SELECT MAX(updated_at) FROM analytics.orders)
) AS S
ON T.order_id = S.order_id

WHEN MATCHED THEN UPDATE SET
    customer_id = S.customer_id,
    status      = S.status,
    amount      = S.amount,
    updated_at  = S.updated_at

WHEN NOT MATCHED THEN INSERT
    (order_id, customer_id, status, amount, updated_at)
VALUES
    (S.order_id, S.customer_id, S.status, S.amount, S.updated_at);
```

## What each part becomes

| In your `MERGE` | In dbt |
| --- | --- |
| `MERGE INTO analytics.orders` | the model's name and location |
| the `USING (...)` subquery | **the model body** — this is your `select` |
| `ON T.order_id = S.order_id` | `unique_key='order_id'` |
| `WHEN MATCHED THEN UPDATE SET ...` | generated — every column, unless you narrow it ([B9](../BACKLOG.md#part-b--write-pattern-archetypes)) |
| `WHEN NOT MATCHED THEN INSERT ...` | generated |
| the `WHERE updated_at > (SELECT MAX...)` | `is_incremental()` block |

Which leaves:

```sql
{{ config(
    materialized='incremental',
    incremental_strategy='merge',
    unique_key='order_id'
) }}

select
    order_id,
    customer_id,
    status,
    amount,
    updated_at
from {{ source('raw', 'orders') }}

{% if is_incremental() %}
  where updated_at > (select max(updated_at) from {{ this }})
{% endif %}
```

The `USING` subquery survives almost verbatim. That's the point: dbt takes over
the mechanics and leaves your logic alone.

## Mapping the `ON` clause properly

The `ON` clause is doing two jobs, and only one of them is `unique_key`.

**Key equality** — the predicates that establish "same row". These become
`unique_key`.

**Everything else** — range bounds, tenant filters, soft-delete guards. These are
*not* key equality and must not go in `unique_key`. They become
`incremental_predicates` ([B12](../BACKLOG.md#part-b--write-pattern-archetypes)).

Splitting a real one:

```sql
ON  T.order_id = S.order_id                              -- key
AND T.region   = S.region                                -- key (composite)
AND T.order_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)  -- NOT a key
```

```sql
unique_key=['order_id', 'region'],
incremental_predicates=[
    "DBT_INTERNAL_DEST.order_date >= date_sub(current_date(), interval 7 day)"
]
```

Note the alias. dbt names the target `DBT_INTERNAL_DEST` and the source
`DBT_INTERNAL_SOURCE`, so predicates you supply must use those names, not your
original `T` and `S`.

## Single key vs composite: not the same code path

A one-column `unique_key` and a multi-column one go through **different branches**
in `default__get_merge_sql`, and they do not behave identically.

Single key routes through `get_merge_unique_key_match`, which BigQuery overrides
to be null-safe when `enable_truthy_nulls_equals_macro` is on.

**A list `unique_key` never calls that macro.** It emits a bare `=` per column:

```jinja
{% for key in unique_key %}
    DBT_INTERNAL_SOURCE.{{ key }} = DBT_INTERNAL_DEST.{{ key }}
{% endfor %}
```

So if any component of a composite key can be `NULL`, the row never matches —
`NULL = NULL` isn't true — and it is re-inserted on every run. The behaviour flag
does not reach this branch.

Your hand-written `MERGE` had exactly the same flaw if it used plain `=`. The
difference is that you could see it. Check for nullable key columns during
conversion, and `coalesce` them in the model if needed. Full detail in
[the expert track](../../expert/03-semantics.md#unique_key-three-branches-two-behaviours).

## No `unique_key` is not "no matching"

If your `MERGE` has an `ON` clause, it has a key, and you must carry it across.
Omitting `unique_key` doesn't mean "match on nothing" — it makes dbt emit
`on FALSE`, so nothing ever matches and **every row inserts**. That's an append,
and it will duplicate silently. See
[the balanced track](../../balanced/02-choosing-a-strategy.md#merge-without-a-unique_key-is-append-only).

## Verify the conversion

Don't trust the mapping — read what dbt generated:

```bash
dbt compile --select orders
```

Then open `target/compiled/<project>/models/orders.sql` and compare it to the
original `MERGE`. You're checking three things:

1. The `ON` clause has the same key predicates, allowing for the alias rename.
2. The `USING` subquery matches your original, filter included.
3. The update column list is what you expect — dbt updates **every** column by
   default, which may be wider than your original `SET`. If so, that's
   [B9](../BACKLOG.md#part-b--write-pattern-archetypes).

Then check row-level parity with [H2](../BACKLOG.md#part-h--proving-correctness)
before retiring the script.

## The clause dbt can't express

If your `MERGE` has `WHEN NOT MATCHED BY SOURCE THEN DELETE`, stop here. dbt's
`merge` strategy emits no such clause, and there is no config that adds one. That
script is asking for `insert_overwrite` semantics or a redesign — see
[B10](../BACKLOG.md#part-b--write-pattern-archetypes).

---

Previous: [A7 · Decide what not to convert](../A-assess/A7-what-not-to-convert.md) ·
Next: [B13 · `DELETE` + `INSERT` → `insert_overwrite`](B13-delete-insert-to-insert-overwrite.md)
