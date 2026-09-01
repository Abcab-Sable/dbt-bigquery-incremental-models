# E10 · System variables

> **Part E — Statement-level translation** · Sourcing: `CRAFT`
> **The question:** my script reads `@@project_id`. What's the dbt equivalent?

`target`, mostly. dbt knows which environment it's running in, and exposes it more
usefully than the session variables do.

## The mapping

| Script | dbt |
| --- | --- |
| `@@project_id` | `{{ target.project }}` |
| `@@dataset_id` | `{{ target.dataset }}` (or `{{ this.schema }}` for the model's own) |
| `@@dataset_project_id` | `{{ target.project }}` |
| `@@time_zone` | Be explicit in SQL — [E9](E9-session-settings.md) |
| `@@error.message` | dbt's failure handling — [C8](../C-structural/C8-exception-handling.md) |
| `@@row_count` | `results[].adapter_response` — [F14](../F-hooks/F14-on-run-start-end.md) |

## `target` is richer

```sql
{{ target.name }}      -- 'dev', 'prod' — the profile target
{{ target.project }}   -- BigQuery project
{{ target.dataset }}   -- default dataset
{{ target.threads }}
```

The important one is `target.name`, which has no script equivalent. Scripts
detected environment by inspecting the project id; dbt tells you directly:

```sql
{% if target.name == 'prod' %}
  ...
{% endif %}
```

Cleaner than string-matching a project name, and it doesn't break when someone
renames a project.

## Don't hardcode what `target` gives you

The failure this prevents:

```sql
-- reads production from every environment
select * from `acme-prod.raw.events`
```

```sql
-- correct
select * from {{ source('raw', 'events') }}
```

If a script used `@@project_id` to build table names dynamically, it was
compensating for not having `ref()` and `source()`. Don't port the mechanism —
replace it with the references, and the problem disappears
([E4](E4-cross-project-references.md)).

## `{{ this }}` for the model's own identity

```sql
{{ this }}              -- fully qualified relation
{{ this.database }}     -- project
{{ this.schema }}       -- dataset
{{ this.identifier }}   -- table name
```

Useful in hooks, and in the watermark pattern
([B6](../B-write-patterns/B6-watermark-filter.md)). Note `this.database` is the
BigQuery *project* — dbt's naming, not BigQuery's
([E4](E4-cross-project-references.md)).

## `invocation_id` and `run_started_at`

Two more with no script equivalent, both useful when converting audit logic:

```sql
{{ invocation_id }}      -- unique per dbt invocation
{{ run_started_at }}     -- UTC timestamp, same for the whole run
```

`run_started_at` is better than `current_timestamp()` for anything that should be
consistent across models in one run — every model gets the same value, which a
per-statement `CURRENT_TIMESTAMP()` does not.

That also makes it deterministic within a run, which matters for
[idempotency](E7-idempotency-meaning.md) and for parity comparisons that would
otherwise differ on every execution.

---

Previous: [E9 · Session settings](E9-session-settings.md) ·
Next: [E11 · Query parameters → vars](E11-query-parameters.md)
