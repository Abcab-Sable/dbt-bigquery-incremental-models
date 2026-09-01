# C5 · `DECLARE` / `SET` scalar variables

> **Part C — Structural archetypes** · Sourcing: `SRC`
> **The question:** my script declares variables at the top. Where do they go?

One of four places, depending on what the variable is for. Getting this right
removes most of a script's apparent complexity.

## The four destinations

| The variable holds | Becomes |
| --- | --- |
| A constant or derived date | An inline expression — [E6](../E-translation/E6-hardcoded-dates.md) |
| Something someone overrides | `var()` — [E11](../E-translation/E11-query-parameters.md) |
| A value read from the target table | A subquery, or `_dbt_max_partition` |
| A value used to build SQL | A Jinja `{% set %}` |

## The common case: a derived date

```sql
DECLARE start_date DATE DEFAULT DATE_SUB(CURRENT_DATE(), INTERVAL 3 DAY);

SELECT ... WHERE event_date >= start_date;
```

The variable exists because SQL scripts need one to reuse a value. Jinja doesn't:

```sql
{% if is_incremental() %}
  where event_date >= date_sub(current_date(), interval 3 day)
{% endif %}
```

If it's used in several places and you want it named, use a Jinja `set` — which is
templating, not SQL, and disappears at compile time:

```sql
{% set lookback_days = 3 %}

...
where event_date >= date_sub(current_date(), interval {{ lookback_days }} day)
```

## Reading from the target: `_dbt_max_partition`

```sql
DECLARE max_date DATE;
SET max_date = (SELECT MAX(event_date) FROM analytics.daily_events);

INSERT ... WHERE event_date > max_date;
```

Two options. The portable one is a subquery:

```sql
{% if is_incremental() %}
  where event_date > (select max(event_date) from {{ this }})
{% endif %}
```

The BigQuery-specific one is `_dbt_max_partition`, which dbt declares for you:

```sql
where event_date > _dbt_max_partition
```

dbt emits the declaration only if the literal string appears in your compiled
code:

```jinja
{%- if '_dbt_max_partition' in compiled_code and language == 'sql' -%}
    declare _dbt_max_partition {{ partition_by.data_type_for_partition() }} default (
      select max({{ partition_by.field }}) from {{ this }}
      where {{ partition_by.field }} is not null
    );
{%- endif -%}
```

Three consequences: it's a **literal substring test**, so building the name
dynamically gives `Undeclared query parameter`; it's **SQL only**; and it
**ignores nulls**. It also requires `partition_by`. Detail in
[the balanced track](../../balanced/04-insert-overwrite.md#_dbt_max_partition).

Prefer the subquery unless you specifically want the scripting variable —
it's clearer and works everywhere.

## Jinja `set` vs SQL variable: the important difference

```sql
{% set cutoff = 3 %}                      -- evaluated at COMPILE time
declare cutoff int64 default 3;           -- evaluated at RUN time
```

A Jinja variable is substituted into the SQL text before BigQuery sees it. A SQL
variable exists during execution.

That matters when the value must come from the data. Jinja cannot see query
results at compile time — unless you use `run_query`, which adds a warehouse
round-trip at every compile and is usually a sign the logic belongs in the model.

**Rule:** if the value is known without querying, use Jinja. If it comes from
data, use a subquery.

## Variables that control flow

```sql
DECLARE row_count INT64;
SET row_count = (SELECT COUNT(*) FROM raw.events);
IF row_count = 0 THEN ... END IF;
```

That's not a variable problem, it's a control-flow problem —
[C6](C6-if-branching.md). Converting the `DECLARE` in isolation won't help.

---

Previous: [C4 · Scripts writing to several tables](C4-fan-out.md) ·
Next: [C6 · `IF` / `ELSEIF` branching](C6-if-branching.md)
