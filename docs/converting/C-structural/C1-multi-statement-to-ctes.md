# C1 · Multi-statement script → CTEs in one model

> **Part C — Structural archetypes** · Sourcing: `CRAFT`
> **The question:** my script builds three temp tables then joins them. Is that one model?

Often yes. If the intermediate results have no independent value, they're CTEs.

## The pattern

```sql
CREATE TEMP TABLE active_users AS
SELECT user_id FROM raw.users WHERE status = 'active';

CREATE TEMP TABLE recent_orders AS
SELECT * FROM raw.orders WHERE order_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY);

CREATE OR REPLACE TABLE analytics.active_customer_orders AS
SELECT o.* FROM recent_orders o JOIN active_users u USING (user_id);
```

Three statements, one output. The first two exist only to feed the third.

## The conversion

```sql
{{ config(materialized='table') }}

with active_users as (
    select user_id from {{ source('raw', 'users') }} where status = 'active'
),

recent_orders as (
    select * from {{ source('raw', 'orders') }}
    where order_date >= date_sub(current_date(), interval 30 day)
)

select o.*
from recent_orders o
join active_users u using (user_id)
```

Each temp table becomes a CTE with the same name. The final statement becomes the
model's `select`. Structure preserved, one statement —
[E1](../E-translation/E1-one-statement-per-model.md).

## When CTEs are right

- The intermediate has **no independent consumer**
- It's **cheap** — a filter, a projection, a small aggregate
- It's used **once** in the final query
- Naming it as a model would add a table nobody queries

That covers most script temp tables. Scripts create them because SQL has no other
way to name a subquery, not because the result matters.

## When they aren't

**Used more than once.** BigQuery may evaluate a CTE per reference rather than
materialising it, so a CTE referenced three times can run three times. If it's
expensive, that's a real cost — make it a model, or `ephemeral`
([C2](C2-ephemeral-models.md)) won't help either since that also inlines.

**Expensive and reused across models.** Then it's a shared intermediate and wants
to be a real table — [C3](C3-separate-models.md).

**Someone queries the temp table's contents.** They can't any more. Check with
[A2](../A-assess/A2-map-dependencies.md) before collapsing it.

**The CTE chain gets long.** Ten CTEs in one model is a model doing too much. The
threshold is judgement, but if you can't hold the whole thing in your head,
readers can't either.

## Keep the names

Resist renaming the CTEs during conversion. Someone will diff your model against
the script, and matching names make that a five-minute job instead of an hour.
Tidy afterwards, separately — [K11](../BACKLOG.md#part-k--anti-patterns).

## `CREATE TEMP TABLE` vs `WITH` isn't semantically neutral

One difference worth knowing: a temp table is **materialised once** and reused; a
CTE may be **re-evaluated per reference**.

If the script's temp table was non-deterministic — a sample, a `LIMIT` without
`ORDER BY`, anything using `RAND()` — the script got one consistent answer and a
CTE might not. That's rare but real, and it presents as intermittent differences
in [H4](../H-verification/H4-column-level-diffing.md).

Where it matters, make it a model instead.

---

Next: [C2 · Ephemeral models](C2-ephemeral-models.md) ·
Back to [the backlog](../BACKLOG.md)
