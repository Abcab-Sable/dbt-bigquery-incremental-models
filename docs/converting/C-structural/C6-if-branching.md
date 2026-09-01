# C6 · `IF` / `ELSEIF` branching

> **Part C — Structural archetypes** · Sourcing: `CRAFT`
> **The question:** my script branches on a condition. Can Jinja do that?

Jinja can branch on things known at **compile** time. It cannot branch on your
data. Which of those your script does decides whether this converts cleanly or
needs a redesign.

## The two kinds of condition

**Compile-time** — environment, config, whether this is an incremental run:

```sql
IF @@dataset_id = 'prod' THEN ...
```

Converts directly to Jinja:

```sql
{% if target.name == 'prod' %}
  ...
{% endif %}
```

**Run-time** — depends on the data:

```sql
DECLARE row_count INT64;
SET row_count = (SELECT COUNT(*) FROM raw.events WHERE event_date = CURRENT_DATE());

IF row_count = 0 THEN
    INSERT INTO ops.alerts VALUES ('no events today');
ELSE
    INSERT INTO analytics.daily_events SELECT ...;
END IF;
```

This cannot become Jinja, because at compile time the count doesn't exist yet.

## Converting compile-time branches

These are the easy ones, and they're common:

```sql
{{ config(materialized='incremental') }}

select ...
from {{ source('raw', 'events') }}
{% if target.name == 'dev' %}
  where event_date >= date_sub(current_date(), interval 7 day)   -- small dev slice
{% endif %}
```

Also useful: `is_incremental()`, `var()`, and `execute` for parse-vs-run
distinctions.

## Converting run-time branches

Three approaches, in order of preference.

**1. Make it set-based.** Most data-conditional branches are a `WHERE` in
disguise:

```sql
-- script
IF (SELECT COUNT(*) FROM staging WHERE valid) > 0 THEN INSERT ... END IF;
```

```sql
-- model: the insert of zero rows is a no-op anyway
select * from {{ ref('staging') }} where valid
```

If the branch guards against inserting nothing, delete it — inserting nothing is
harmless. Though check [B14](../B-write-patterns/B14-when-the-range-can-empty.md)
first, because under `insert_overwrite` "nothing" has a meaning.

**2. Make it a test.** A branch that alerts on bad data is a test:

```sql
-- tests/events_present_today.sql
select 1
from (select count(*) as n from {{ ref('daily_events') }}
      where event_date = current_date())
where n = 0
```

Better than the script's version: it runs in `dbt build`, blocks downstream
models, and reports through the same channel as every other failure.
See [H12](../H-verification/H12-tests-from-guarantees.md).

**3. Move it to the orchestrator.** If the branch decides *whether to run at
all*, that's a scheduling decision. Your orchestrator checks the condition and
invokes dbt or doesn't. Don't simulate it inside a model —
[K5](../BACKLOG.md#part-k--anti-patterns).

## `run_query` is the tempting wrong answer

You *can* query at compile time:

```sql
{% set result = run_query('select count(*) from raw.events') %}
{% if result.columns[0].values()[0] > 0 %}
```

It works. It also means every compile — including `dbt compile`, `dbt docs
generate`, and CI parsing — hits the warehouse. On a large project that's slow
and expensive, and the dependency is invisible.

Reserve it for genuine metaprogramming, like generating a column list from
`INFORMATION_SCHEMA`. Not for business logic.

## The honest outcome

If the script's branching is genuinely procedural — different tables written
depending on data, multi-way logic with side effects — the conversion is a
redesign, not a translation.

That's fine, but recognise it early. It's an [A8](../A-assess/A8-estimate-risk.md)
high-difficulty signal, and forcing it into Jinja produces something worse than
what you started with.

---

Previous: [C5 · `DECLARE` / `SET` scalar variables](C5-declare-set-variables.md) ·
Next: [C7 · `WHILE` / `LOOP` iteration](C7-loops.md)
