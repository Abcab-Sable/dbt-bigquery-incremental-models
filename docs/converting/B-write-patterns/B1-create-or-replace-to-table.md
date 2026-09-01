# B1 · `CREATE OR REPLACE TABLE ... AS SELECT` → `materialized='table'`

> **Part B — Write-pattern archetypes** · Sourcing: `SRC`
> **The question:** the simplest possible conversion. What's involved?

The one-to-one case. Your script defines a table by a query and rebuilds it whole;
so does the `table` materialization.

## Before and after

```sql
CREATE OR REPLACE TABLE analytics.customer_summary AS
SELECT customer_id, COUNT(*) AS orders, SUM(amount) AS total
FROM raw.orders
GROUP BY customer_id;
```

```sql
{{ config(materialized='table') }}

select customer_id, count(*) as orders, sum(amount) as total
from {{ source('raw', 'orders') }}
group by customer_id
```

The `CREATE OR REPLACE TABLE ... AS` disappears; the `SELECT` is the model. That
is genuinely the whole conversion.

## Don't make it incremental

The commonest mistake here is deciding a big table "should" be incremental. Read
[A7](../A-assess/A7-what-not-to-convert.md) first. A `table` model is:

- always correct, because it recomputes from source every run
- immune to every silent failure in this documentation
- trivially idempotent

You give all that up for a cost saving. Make that trade deliberately, when the
cost is real, not because incremental feels more advanced.

## What to carry across

**Partitioning and clustering.** If the script's `CREATE` had them, they go in
config or you'll silently drop them:

```sql
{{ config(
    materialized='table',
    partition_by={'field': 'order_date', 'data_type': 'date'},
    cluster_by=['customer_id']
) }}
```

Check the existing DDL rather than trusting the script —
[A5](../A-assess/A5-hidden-state.md):

```sql
select ddl from `project.analytics.INFORMATION_SCHEMA.TABLES`
where table_name = 'customer_summary';
```

**Table options.** Expiration, labels, description — [D7](../BACKLOG.md#part-d--data-movement-ddl-and-metadata).

**Grants.** Into the `grants` config, not a hook —
[D10](../BACKLOG.md#part-d--data-movement-ddl-and-metadata).

## The one behavioural difference

`CREATE OR REPLACE TABLE` is atomic — readers see the old table until the new one
is ready.

dbt's `table` materialization on BigQuery goes through `bq_create_table_as` and is
also a replace, so this generally holds. But if the partitioning or clustering
spec changes, `is_replaceable` returns false and dbt **drops and recreates**
instead. See
[the balanced track](../../balanced/01-how-the-materialization-runs.md#full-refresh-can-silently-become-a-drop).

So: steady-state runs behave like your script. The run where you *change* the
partitioning does not, and there's a window where the table doesn't exist.

## Verify

```bash
dbt run --select customer_summary
```

Then [H2](../H-verification/H2-row-count-parity.md). For a full-rebuild model
this should match exactly on the first attempt — if it doesn't, the difference is
in your `select`, not in the materialization.

---

Previous: [E8 · Idempotency: proving it](../E-translation/E8-idempotency-proving.md) ·
Next: [B2 · `CREATE TABLE IF NOT EXISTS` bootstrap](B2-create-if-not-exists.md)
