# C10 · `EXECUTE IMMEDIATE` and dynamic SQL

> **Part C — Structural archetypes** · Sourcing: `CRAFT`
> **The question:** my script builds SQL as a string and runs it. Does Jinja replace that?

Often yes — Jinja is a templating language, which is exactly what the script was
hand-rolling. The question is whether the inputs to the template are known at
compile time.

## The good case: a known list

```sql
FOR region IN (SELECT 'eu' UNION ALL SELECT 'us') DO
    EXECUTE IMMEDIATE FORMAT("""
        INSERT INTO analytics.events SELECT * FROM raw.events_%s
    """, region);
END FOR;
```

The list is fixed, so Jinja generates the SQL directly:

```sql
{% set regions = ['eu', 'us', 'apac'] %}

{% for region in regions %}
select '{{ region }}' as region, * from {{ source('raw', 'events_' ~ region) }}
{% if not loop.last %}union all{% endif %}
{% endfor %}
```

Better than the original: it's one statement, it's visible in compiled output,
and the sources are real DAG edges instead of strings.

## The awkward case: the list comes from data

```sql
EXECUTE IMMEDIATE (
    SELECT FORMAT("SELECT %s FROM raw.events", STRING_AGG(column_name))
    FROM INFORMATION_SCHEMA.COLUMNS WHERE table_name = 'events'
);
```

Jinja can do this with `run_query`, but it costs you a warehouse round-trip on
every compile:

```sql
{% set cols = run_query("select column_name from ... where table_name = 'events'") %}
{% if execute %}
  select {{ cols.columns[0].values() | join(', ') }} from {{ source('raw','events') }}
{% endif %}
```

The `{% if execute %}` guard is mandatory — during parsing, `run_query` returns
nothing and the model must still compile.

Use it for genuine metaprogramming (generating a column list, pivoting on a known
set of values). Don't use it for business logic. Every `dbt compile`, `dbt docs
generate`, and CI parse now queries the warehouse, and the dependency is invisible
in the DAG.

## Prefer `adapter.get_columns_in_relation`

For the specific and common case of "I need the column list", there's a
first-class API that doesn't need raw SQL:

```sql
{% set cols = adapter.get_columns_in_relation(ref('events')) %}

select
{% for c in cols %}
    {{ c.name }}{{ ',' if not loop.last }}
{% endfor %}
from {{ ref('events') }}
```

Same result, cleaner, and it goes through the adapter's caching.

## Dynamic table names defeat lineage

The most damaging part of dynamic SQL in a conversion:

```sql
EXECUTE IMMEDIATE FORMAT("INSERT INTO %s SELECT ...", target_table);
```

A table name that only exists as a string is invisible to
[E5](../E-translation/E5-finding-hardcoded-names.md)'s greps *and* to dbt. If you
port this shape, you get a model with no edges and intermittent ordering bugs.

The check that catches it is BigQuery's own record of what was read:

```sql
select referenced_tables
from `region-eu`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
where creation_time > timestamp_sub(current_timestamp(), interval 7 day)
  and query like '%your_script_marker%'
order by creation_time desc limit 5;
```

That tells you what the dynamic SQL actually touched, which is what you need to
write the `ref()`s.

## When it stays a `run-operation`

Some dynamic SQL isn't building a model at all — it's administrative. Dropping
old partitions, granting across a list of datasets, applying labels in bulk.

That's a macro invoked with `dbt run-operation`:

```sql
{% macro drop_old_partitions(dataset, days) %}
  {% set sql %}...{% endset %}
  {% do run_query(sql) %}
{% endmacro %}
```

```bash
dbt run-operation drop_old_partitions --args '{dataset: analytics, days: 90}'
```

Legitimate, and the right home for maintenance work. **Not** a scheduler and not
a substitute for models — [K4](../K-antipatterns/K4-run-operation-as-scheduler.md).

## Ask first: does it need to be dynamic?

Scripts use dynamic SQL for reasons that often evaporate:

- **Iterating tables** ⇒ usually wildcards ([D4](../D-data-movement/D4-wildcard-tables.md))
  or a `union all`
- **A parameterised table name** ⇒ usually a `var`, or separate models
- **Conditional columns** ⇒ usually a fixed superset with nulls
- **Sharded tables** ⇒ a partitioned table ([D5](../D-data-movement/D5-sharded-tables.md))

The last one is the big one. A great deal of dynamic SQL in BigQuery scripts
exists purely to loop over `events_20260831`-style shards, and converting to a
partitioned table removes the need entirely.

---

Previous: [C9 · `BEGIN TRANSACTION` / `COMMIT`](C9-transactions.md) ·
Next: [C11 · `CREATE TEMP FUNCTION`](C11-temp-functions.md)
