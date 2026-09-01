# B2 · `CREATE TABLE IF NOT EXISTS` bootstrap → letting dbt own creation

> **Part B — Write-pattern archetypes** · Sourcing: `SRC`
> **The question:** my script creates the table if it's missing, then inserts. What happens to the create?

It goes away. dbt creates the table, and the bootstrap block is the part of your
script that existed only because nothing else would do it.

## The pattern

```sql
CREATE TABLE IF NOT EXISTS analytics.daily_events (
    event_date DATE,
    user_id    INT64,
    event_count INT64
)
PARTITION BY event_date;

INSERT INTO analytics.daily_events
SELECT event_date, user_id, COUNT(*) FROM raw.events
WHERE event_date = CURRENT_DATE()
GROUP BY 1, 2;
```

## What each half becomes

The `CREATE` is **schema and physical layout**, and it moves into config:

```sql
{{ config(
    materialized='incremental',
    incremental_strategy='insert_overwrite',
    partition_by={'field': 'event_date', 'data_type': 'date', 'granularity': 'day'}
) }}
```

The `INSERT` is the model. The column list is implied by your `select`.

```sql
select event_date, user_id, count(*) as event_count
from {{ source('raw', 'events') }}
{% if is_incremental() %}
  where event_date = current_date()
{% endif %}
group by event_date, user_id
```

dbt takes the "if not exists" branch automatically: the materialization checks
`existing_relation is none` and runs a full `CREATE TABLE AS` when the table
isn't there. Your model's first run *is* the bootstrap.

## Column types are now inferred

The explicit types in the `CREATE` are gone, and BigQuery infers them from your
`select`. Mostly fine, occasionally not:

- `COUNT(*)` gives `INT64` — matches
- A literal `NULL` gives `INT64` unless you cast it
- Numeric precision may differ from a declared `NUMERIC(10,2)`

If types matter, cast explicitly in the model. If they matter *a lot* — a
contract with a downstream consumer — use dbt's `contract` config to enforce
them rather than hoping.

Capture the original types before you start
([A9](../A-assess/A9-correctness-baseline.md)) so you can compare:

```sql
select column_name, data_type
from `project.analytics.INFORMATION_SCHEMA.COLUMNS`
where table_name = 'daily_events'
order by ordinal_position;
```

## The bootstrap tells you about hidden state

A script that creates its own table usually predates any managed infrastructure.
Look at what else that `CREATE` specified — expiration, clustering, description,
options — because all of it must move into config or it's lost on the first
`--full-refresh`. That's [A5](../A-assess/A5-hidden-state.md).

## Watch for divergence

The `IF NOT EXISTS` means the `CREATE` hasn't run in years. If someone later
altered the live table — added a column, changed clustering — the script's
`CREATE` block no longer describes reality.

**Trust the live DDL, not the script.** Read it from `INFORMATION_SCHEMA` and
build your config from that.

---

Previous: [B1 · `CREATE OR REPLACE TABLE` → `table`](B1-create-or-replace-to-table.md) ·
Next: [B3 · `CREATE VIEW` → `view`](B3-create-view.md)
