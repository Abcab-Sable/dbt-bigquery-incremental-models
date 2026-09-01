# E11 · Query parameters (`@param`) → vars

> **Part E — Statement-level translation** · Sourcing: `CORE`
> **The question:** my scheduled query takes parameters. How are those passed?

As dbt vars. The mechanism differs in one important way: parameters are bound at
**run** time, vars are substituted at **compile** time.

## The pattern

A scheduled query with `@run_date`, or a script invoked with parameters:

```sql
SELECT * FROM raw.events WHERE event_date = @run_date;
```

```bash
bq query --parameter='run_date:DATE:2026-08-31' < query.sql
```

## The conversion

```sql
{% set run_date = var('run_date', none) %}

select * from {{ source('raw', 'events') }}
where event_date = {% if run_date %} '{{ run_date }}' {% else %} current_date() {% endif %}
```

```bash
dbt run --select daily_events --vars '{run_date: 2026-08-31}'
```

Always give `var()` a default. A var with no default and no value raises at
compile time, so a scheduled run that forgets it fails rather than defaulting
sensibly.

## Compile-time, not bind-time

The difference that matters.

A query parameter is **bound** — BigQuery receives `@run_date` and a value
separately. A dbt var is **substituted** — the value is written into the SQL text
before BigQuery sees it.

Consequences:

**No injection protection.** `--vars '{table: "x; drop table y"}'` becomes SQL.
Parameters were safe by construction; vars are not. Don't build vars from
untrusted input, and quote/cast them explicitly.

**Compiled SQL differs per invocation.** Two runs with different vars produce
different compiled SQL, which affects comparison and caching. The compiled
artefact is only meaningful alongside the vars it was built with.

**Type handling is yours.** A parameter had a declared type; a var is text. Cast
explicitly:

```sql
where event_date = date('{{ var("run_date") }}')
```

## Where to set them

**In `dbt_project.yml`** for stable defaults:

```yaml
vars:
  lookback_days: 3
  region: 'eu'
```

**On the command line** for overrides:

```bash
dbt run --vars '{lookback_days: 30}'
```

**From the environment**, for anything secret or deployment-specific:

```yaml
vars:
  gcs_bucket: "{{ env_var('GCS_BUCKET', 'acme-dev-bucket') }}"
```

Secrets belong in `env_var`, never in `dbt_project.yml`, and never in a var passed
on a command line that ends up in shell history or CI logs.

## Microbatch is the better fit for date parameters

If the parameter is a **date being iterated for a backfill**, vars are the manual
version of something dbt does properly:

```sql
{{ config(
    materialized='incremental',
    incremental_strategy='microbatch',
    event_time='event_date',
    batch_size='day',
    begin='2024-01-01'
) }}
```

```bash
dbt run --select daily_events --event-time-start 2024-01-01 --event-time-end 2024-06-30
```

dbt handles batch boundaries, ordering, and retry. See
[the microbatch page](../../balanced/05-microbatch.md) — the boundary maths is
pinned and documented, including the `lookback` increment when a checkpoint lands
exactly on a batch line.

For a genuine backfill loop, this beats shelling out with vars in a bash `for`.

## Don't over-parameterise

The opposite failure: every literal becomes a var "for flexibility", and you get a
model nobody can read with defaults nobody can find. That's
[K7](../BACKLOG.md#part-k--anti-patterns).

Parameterise what someone actually overrides. Everything else is a constant, and
constants are clearer inline — [E6](E6-hardcoded-dates.md).

---

Previous: [E10 · System variables](E10-system-variables.md) ·
Next: [E12 · Cost controls](E12-cost-controls.md)
