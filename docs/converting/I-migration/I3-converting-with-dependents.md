# I3 · Converting one script while others depend on its output

> **Part I — Migration strategy** · Sourcing: `CRAFT`
> **The question:** three unconverted scripts read this table. Can I still convert it?

Yes — as long as the model writes to the **same table** the script did. The
dependents don't care what produced it.

## Write to the same place

```sql
{{ config(
    materialized='incremental',
    schema='analytics',
    alias='daily_events'
) }}
```

Now the model produces `analytics.daily_events`, exactly what the script produced.
The three scripts reading it keep working, unaware anything changed.

`alias` matters if your project prefixes or transforms model names — the
downstream scripts have the old name hardcoded, and it must not move.

## The custom schema catch

dbt's default schema generation prefixes the target schema onto the configured
one, so `schema='analytics'` in a project targeting `analytics` can yield
`analytics_analytics`.

Check what you actually get:

```bash
dbt compile --select daily_events
grep -i "insert into\|merge into" target/compiled/*/models/**/daily_events.sql
```

If it's wrong, override `generate_schema_name` — but do it deliberately, since it
affects every model in the project.

Getting this wrong writes to a new table while the old one goes stale, and the
dependent scripts keep reading the stale one. Nothing errors.

## The dependency is now invisible to dbt

The three scripts read the table but don't `ref()` it, so dbt doesn't know they
exist. It can't warn you before a breaking change, and the lineage graph is
incomplete.

Record them as exposures:

```yaml
exposures:
  - name: legacy_user_summary_script
    type: application
    description: >
      user_summary.sql (scheduled query, 04:00 UTC) reads analytics.daily_events.
      Not yet converted — DATA-2291.
    depends_on:
      - ref('daily_events')
    owner:
      name: Data Platform
      email: data@acme.com
```

Now the dependency is in the DAG and the docs, which is what stops someone
changing the model's columns without realising a script consumes them —
[I5](I5-notifying-consumers.md).

## Don't change the interface yet

While unconverted dependents exist, treat the table's shape as frozen:

- Don't rename or drop columns
- Don't change types
- Don't change the partition column
- Don't change the grain

All of those are safe *after* the dependents are converted. Doing them at the same
time as the conversion is [K11](../K-antipatterns/K11-convert-and-optimise.md),
and here it breaks other people's jobs rather than just confusing the diff.

Additive changes are fine. New columns don't break a script that selects
explicitly — though one doing `SELECT *` into a fixed insert list will break, so
check that too.

## Timing still matters

The dependents run on a schedule assuming the table is ready by a certain time. If
your model runs later than the script did, they read yesterday's data.

Keep the schedule roughly where it was until the dependents are converted, then
consolidate — [G2](../G-scheduling/G2-consolidating-schedules.md). Consolidating
while external readers still have timing assumptions is how a conversion breaks
something that isn't in the project.

## Convert the dependents next

Each dependent you convert removes an exposure and turns a hardcoded table name
into a `ref()`. Prioritise them for that reason — every one converted reduces the
constraints on the model you just built.

---

Previous: [I2 · The strangler pattern](I2-strangler-pattern.md) ·
Next: [I4 · Dual-write during cutover](I4-dual-write.md)
