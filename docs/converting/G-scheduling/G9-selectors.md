# G9 · Selectors: `--select` and `--exclude`

> **Part G — Scheduling, parameters, backfills** · Sourcing: `CORE✓`
> **The question:** how do I run part of the project?

Selectors. They replace the per-script granularity you lose by consolidating into
one `dbt build`, and they're the thing that makes consolidation practical.

## Graph operators

| Selector | Runs |
| --- | --- |
| `daily_events` | that model |
| `+daily_events` | it and all ancestors |
| `daily_events+` | it and all descendants |
| `+daily_events+` | ancestors, it, descendants |
| `2+daily_events` | two generations of ancestors |
| `@daily_events` | it, descendants, and their ancestors |

`+model` is the direct replacement for "run this script and everything it needed
first" — [G1](G1-cron-to-dbt-build.md).

## Selection methods

dbt-core defines these in `MethodName`:

```
fqn · tag · group · access · source · path · file · package · config
test_name · test_type · resource_type · state · exposure · metric
result · source_status · version · semantic_model · saved_query
unit_test · selector
```

The ones that earn their keep during and after a conversion:

```bash
dbt build --select tag:daily                    # by cadence
dbt build --select path:models/marts            # by folder
dbt build --select source:raw+                  # everything from one source
dbt build --select config.materialized:incremental
dbt build --select result:error+ --state ./prev # rerun failures and dependants
```

`config.materialized:incremental` is genuinely useful post-conversion — it's how
you run just the models with the failure modes this documentation is about.

## Combining

**Union** — space-separated:

```bash
dbt build --select tag:daily tag:hourly
```

**Intersection** — comma-separated:

```bash
dbt build --select tag:daily,config.materialized:incremental
```

**Exclusion:**

```bash
dbt build --select tag:daily --exclude tag:expensive
```

Intersection is the one people miss, and it's how you express "daily models that
are incremental" without tagging that combination explicitly.

## During conversion

Selectors are how you run the new model without running everything:

```bash
# just the converted model and what it needs
dbt build --select +daily_events

# everything downstream, to check nothing broke
dbt build --select daily_events+

# the model and its tests only
dbt build --select daily_events
```

And how you shadow it — [H5](../H-verification/H5-shadow-mode.md):

```bash
dbt build --select daily_events --target shadow
```

## Save the complicated ones

If a selector gets long, name it in `selectors.yml`:

```yaml
selectors:
  - name: daily_marts
    definition:
      union:
        - method: tag
          value: daily
        - method: path
          value: models/marts
```

```bash
dbt build --selector daily_marts
```

Version-controlled, reviewable, and it stops the same expression being retyped
slightly differently in three cron entries.

## Don't rebuild the script's granularity

Tempting after converting forty scripts to create forty `--select` invocations,
one per old script. That recreates the problem consolidation solved: ordering
back in the scheduler, and margin between jobs.

Group by cadence, use `+` for dependencies, and let dbt order it —
[G2](G2-consolidating-schedules.md).

---

Previous: [G8 · Late-arriving data](G8-late-arriving-data.md) ·
Next: [G10 · State-based selection](G10-state-selection.md)
