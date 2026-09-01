# C2 · Multi-statement script → ephemeral models

> **Part C — Structural archetypes** · Sourcing: `CORE✓`
> **The question:** what does `materialized='ephemeral'` actually do?

It turns a model into a CTE injected into whatever references it. No table, no
view, nothing in the warehouse — but it gets a name, a place in the DAG, and
tests.

## The mechanism

From dbt-core's `compilation.py`, ephemeral models are compiled and injected as
CTEs with a prefix:

```python
new_cte_name = self.add_ephemeral_prefix(cte_name)
```

Producing SQL like:

```sql
with __dbt__cte__ephemeral as (select * from table),
     __dbt__cte__events as (select id, type from events),
select * from __dbt__cte__events
```

The `__dbt__cte__` prefix is how you spot ephemeral models in compiled output.
Compilation happens under a lock so only one thread compiles each ephemeral
model, and referencing a non-ephemeral node as a CTE raises `DbtInternalError`.

## Ephemeral vs a plain CTE

Both end up as a CTE in the final query. The difference is everything around it:

| | CTE in the model | Ephemeral model |
| --- | --- | --- |
| Appears in the DAG | No | **Yes** |
| Can be `ref()`d by other models | No | **Yes** |
| Can have tests and docs | No | **Yes** |
| Exists in the warehouse | No | No |
| Reusable across models | No | **Yes** |

So: use a CTE when the intermediate belongs to one model. Use ephemeral when
**several models need the same intermediate** and you don't want a table.

## The conversion case

A script's temp table used by two later statements, where those statements become
two models:

```sql
-- models/staging/stg_active_users.sql
{{ config(materialized='ephemeral') }}

select user_id, signup_date
from {{ source('raw', 'users') }}
where status = 'active'
```

```sql
-- models/marts/active_orders.sql
select o.* from {{ ref('orders') }} o
join {{ ref('stg_active_users') }} u using (user_id)
```

Both consumers get the same definition, tested once, with no extra table.

## The costs

**The SQL is inlined into every consumer.** Ten models referencing one ephemeral
model means that query text appears ten times, and runs ten times. Ephemeral
saves storage, not compute.

**Compiled SQL gets hard to read.** Nested ephemeral models produce deeply
stacked CTEs with generated names. Debugging a four-level ephemeral chain in
`target/compiled/` is genuinely unpleasant.

**You can't query it.** Nothing exists to `select *` from during debugging. You
have to compile a consumer and pull the CTE out.

**Not all tests work.** Tests on an ephemeral model are inlined too; some test
types behave differently, and `--store-failures` has nothing to point at.

## When to use a table instead

If the intermediate is expensive and referenced more than once, ephemeral is the
wrong answer — you pay for it every time. Materialise it as a `view` or `table`
and let consumers read the result once.

The rule of thumb: **ephemeral for cheap shared logic, table for expensive shared
results.**

## Don't chain them deeply

One level is fine. Two is tolerable. Beyond that the compiled SQL becomes
unreadable and the cost multiplication gets hard to reason about. That's
[K9](../K-antipatterns/K9-ephemeral-overuse.md).

If a script had a five-deep temp table chain, that's usually a sign the pipeline
wants real intermediate tables — [C3](C3-separate-models.md).

---

Previous: [C1 · Multi-statement → CTEs](C1-multi-statement-to-ctes.md) ·
Next: [C3 · Separate models](C3-separate-models.md)
