# F12 · post-hook: table options and metadata

> **Part F — Hooks** · Sourcing: `CRAFT`
> **The question:** my script sets a description, labels, and an expiry. Where do those go?

Mostly config; some genuinely need a hook. The split isn't obvious, so here it is
explicitly.

## What's already config

| Script sets | dbt config | Hook needed? |
| --- | --- | --- |
| `partition_by` | `partition_by` | No |
| `cluster by` | `cluster_by` | No |
| `partition_expiration_days` | `partition_expiration_days` | No |
| `require_partition_filter` | `require_partition_filter` | No |
| Table/column description | `persist_docs` + `description` in YAML | No |
| Grants | `grants` — [F11](F11-grants-vs-post-hook.md) | No |
| Labels | — | **Yes** |
| `expiration_timestamp` | — | **Yes** |
| `friendly_name` | — | **Yes** |

Reach for config first. Every row you can move out of a hook is one fewer thing
that runs unconditionally on every build.

## The description trap

`persist_docs` runs **after** post-hooks ([F4](F4-where-hooks-run.md)). So if you
set a description in a hook *and* have `persist_docs` enabled, dbt overwrites
yours.

```jinja
{{ run_hooks(post_hooks) }}       ← your ALTER ... SET OPTIONS(description=...)
...
{% do persist_docs(target_relation, model) %}   ← dbt overwrites it
```

Use one or the other. `persist_docs` with descriptions in YAML is the better
choice — it's the same text that appears in the docs site, so they can't drift:

```yaml
models:
  - name: daily_events
    description: Daily event counts by user.
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

## Labels genuinely need a hook

No dbt config covers BigQuery labels, and they're often mandated for cost
attribution:

```sql
{{ config(post_hook="""
    alter table {{ this }} set options(
        labels = [('team', 'analytics'), ('cost-centre', 'cc-4471')]
    )
""") }}
```

For a whole folder, put it in `dbt_project.yml` so it isn't repeated per model:

```yaml
models:
  my_project:
    marts:
      +post_hook: "{{ apply_standard_labels(this) }}"
```

```sql
{% macro apply_standard_labels(relation) %}
  alter table {{ relation }} set options(
      labels = [('team', 'analytics'), ('managed-by', 'dbt')]
  )
{% endmacro %}
```

A macro rather than an inline string: testable, and the quoting stays sane
([F2](F2-hook-rendering.md)).

## `SET OPTIONS` replaces, it doesn't merge

Setting `labels` replaces the whole label set. If something else applies labels —
Terraform, a governance process — your hook removes theirs.

Check before you convert:

```sql
select option_name, option_value
from `project.dataset.INFORMATION_SCHEMA.TABLE_OPTIONS`
where table_name = 'daily_events';
```

Anything there that your config and hooks don't reproduce will be lost on the
next `--full-refresh`. That's [A5](../A-assess/A5-hidden-state.md) territory.

## Idempotent and cheap

`ALTER TABLE ... SET OPTIONS` is idempotent — setting the same options twice is a
no-op — and it's metadata-only, so it doesn't scan. This is close to the ideal
post-hook: fast, safe to repeat, and genuinely about the relation.

## Prefer config where it exists

Expiration is the common mistake. This works:

```sql
post_hook="alter table {{ this }} set options(partition_expiration_days=90)"
```

But this is better:

```sql
{{ config(partition_expiration_days=90) }}
```

Applied as part of table creation rather than after it, visible in the manifest,
and no extra statement on every run.

---

Previous: [F11 · post-hook vs the `grants` config](F11-grants-vs-post-hook.md) ·
Next: [F13 · post-hook: audit rows](F13-post-hook-audit-rows.md)
