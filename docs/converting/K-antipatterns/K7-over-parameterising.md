# K7 · Over-parameterising with vars

> **Part K — Anti-patterns** · Sourcing: `CRAFT`
> **The question:** shouldn't I make things configurable while I'm converting?

Rarely. A var for something nobody overrides is indirection with no payoff, and
the cost lands on every future reader.

## The shape

```sql
{% set schema = var('target_schema', 'analytics') %}
{% set lookback = var('lookback_days', 3) %}
{% set date_col = var('date_column', 'event_date') %}
{% set threshold = var('min_count', 1) %}
{% set exclude_test = var('exclude_test_users', true) %}

select {{ date_col }}, user_id, count(*) as n
from {{ source(var('source_name', 'raw'), var('source_table', 'events')) }}
where {{ date_col }} >= date_sub(current_date(), interval {{ lookback }} day)
  {% if exclude_test %} and not is_test_user {% endif %}
group by 1, 2
having count(*) >= {{ threshold }}
```

Seven vars. To know what this model does you need the model, `dbt_project.yml`,
and whatever the scheduler passes.

## The costs

**You can't read it.** The SQL is now a template of a query rather than a query.

**Defaults are elsewhere.** `var('lookback_days', 3)` might be overridden in
`dbt_project.yml`, or on the command line. Three places to check.

**It varies by invocation.** Two runs produce different compiled SQL, so
`target/compiled/` is only meaningful alongside the vars used — which makes
[parity checking](../H-verification/H3-checksum-parity.md) harder.

**Injection surface.** Vars are substituted, not bound. `var('date_column')` goes
straight into the SQL — [E11](../E-translation/E11-query-parameters.md).

**Untested combinations.** Five booleans is thirty-two behaviours, of which you
have tested one.

## The test

> **Has anyone actually overridden this, or would they?**

If no: it's a constant. Write it inline, where it can be read.

Legitimate vars are few:

| Var | Why |
| --- | --- |
| `backfill_start` / `backfill_end` | Genuinely passed for backfills — [G7](../G-scheduling/G7-backfill-partition-ranges.md) |
| Environment-varying config | Bucket names, project ids — [E4](../E-translation/E4-cross-project-references.md) |
| A feature flag with a removal date | Temporary, with a ticket |

Nearly everything else is a constant.

## Where it comes from

The script had `DECLARE` at the top, so the converted model gets `var()` at the
top. But a script variable existed because SQL scripts need one to reuse a value —
not because anyone changed it
([C5](../C-structural/C5-declare-set-variables.md)).

The tell in the original is a commented-out alternative:

```sql
DECLARE start_date DATE DEFAULT DATE_SUB(CURRENT_DATE(), INTERVAL 3 DAY);
-- DECLARE start_date DATE DEFAULT '2024-01-01';  -- backfill
```

**That** is a real parameter — someone edits it. Everything else in the `DECLARE`
block is a constant.

## Target-based branching is different

```sql
{% if target.name == 'ci' %}
  where event_date >= date_sub(current_date(), interval 3 day)
{% endif %}
```

Not a var, and legitimate — it's environment behaviour, and it's what makes CI
affordable ([G10](../G-scheduling/G10-state-selection.md)).

## Prefer configs over vars

Real dbt configs beat home-made vars:

```sql
-- rather than var('materialization', 'table')
{{ config(materialized='table') }}
```

Configs are typed, documented, settable per folder in `dbt_project.yml`, and
visible in the manifest. A hand-rolled var is none of those.

## Removing them later

Grep for the var, check nothing passes it, inline the default, delete it. Cheap —
which is the argument for not adding it speculatively in the first place.

---

Previous: [K6 · Porting the bug faithfully](K6-porting-the-bug.md) ·
Next: [K8 · One model per statement](K8-one-model-per-statement.md)
