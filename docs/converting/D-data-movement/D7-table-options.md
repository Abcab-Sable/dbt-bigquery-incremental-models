# D7 · Expiration, labels, and description

> **Part D — Data movement, DDL, metadata** · Sourcing: `CRAFT`
> **The question:** which table options are config and which need a hook?

Split covered in [F12](../F-hooks/F12-post-hook-table-options.md); this is the
conversion angle — finding what the live table has and making sure none of it is
lost.

## Find what's actually set

Not what the script says. What the table has:

```sql
select option_name, option_value
from `project.analytics.INFORMATION_SCHEMA.TABLE_OPTIONS`
where table_name = 'daily_events';
```

Everything in that result must be reproduced by your config or a hook, or it
disappears on the first `--full-refresh`. This is
[A5](../A-assess/A5-hidden-state.md) in its most concrete form.

## The split

| Option | Where |
| --- | --- |
| `partition_expiration_days` | `config(partition_expiration_days=90)` |
| `require_partition_filter` | `config(require_partition_filter=true)` |
| `description` | YAML `description` + `persist_docs` |
| `expiration_timestamp` | post-hook |
| `labels` | post-hook |
| `friendly_name` | post-hook |

Prefer config. Config is applied at creation, visible in the manifest, and costs
no extra statement per run.

## Description: use `persist_docs`, not a hook

`persist_docs` runs **after** post-hooks, so if you set a description in a hook
*and* enable `persist_docs`, dbt overwrites yours
([F4](../F-hooks/F4-where-hooks-run.md)).

Put it in YAML instead, where it's the same text the docs site shows:

```yaml
models:
  - name: daily_events
    description: Daily event counts by user, one row per user per day.
    columns:
      - name: event_date
        description: UTC date the events occurred on.
```

```yaml
models:
  my_project:
    +persist_docs:
      relation: true
      columns: true
```

Now the description can't drift from the documentation, which the script's
inline text always could.

## Labels: a hook, applied broadly

No config covers them, and they're usually mandated for cost attribution. Define
once at project level rather than per model:

```yaml
models:
  my_project:
    +post_hook: "{{ apply_standard_labels(this) }}"
```

```sql
{% macro apply_standard_labels(relation) %}
  alter table {{ relation }} set options(
      labels = [('team', 'analytics'), ('managed-by', 'dbt')]
  )
{% endmacro %}
```

**`SET OPTIONS` replaces the whole label set.** If Terraform or a governance
process also applies labels, your hook removes theirs. Check the current values
first — that's what the `TABLE_OPTIONS` query above is for.

## Expiration: check what it's protecting

Two different things get called expiration:

**`partition_expiration_days`** — old partitions drop automatically. A retention
policy, and it belongs in config.

**`expiration_timestamp`** — the whole table disappears at a point in time.
Usually a leftover from a temporary table that became permanent. Find out which
before reproducing it; carrying forward an accidental expiry is a bad outcome, and
so is dropping a deliberate one.

If the script set `partition_expiration_days` and you miss it, partitions stop
being cleaned up and storage grows quietly. That won't show up in any parity
check — it's [J8](../J-operating/J8-partition-growth.md) territory.

## Record what you deliberately dropped

Some options shouldn't come across — an expiry that was a mistake, a label scheme
that's been replaced. Note the decision:

```
Options on analytics.daily_events at baseline:
  partition_expiration_days = 90    → carried to config
  labels = [(team, analytics)]      → carried to post-hook
  friendly_name = 'daily events v2' → DROPPED, obsolete naming from 2023
  description = 'TODO'              → DROPPED, replaced with real YAML description
```

Otherwise the next person can't tell a deliberate omission from an oversight.

---

Previous: [D6 · Partitioning and clustering DDL](D6-partitioning-ddl.md) ·
Next: [D8 · `ALTER TABLE ADD COLUMN` migrations](D8-add-column-migrations.md)
