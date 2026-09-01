# F15 · Hooks and the temp relation

> **Part F — Hooks** · Sourcing: `SRC`
> **The question:** can a post-hook see the temp table dbt built?

Sometimes. And "sometimes" is the reason not to build anything on it.

## The window exists

From the BigQuery incremental materialization:

```jinja
{{ run_hooks(post_hooks) }}                      ← post-hooks run here

{% set target_relation = this.incorporate(type='table') %}
{% do apply_grants(...) %}
{% do persist_docs(...) %}

{%- if tmp_relation_exists -%}
  {{ adapter.drop_relation(tmp_relation) }}      ← temp table dropped here
{%- endif -%}
```

The drop happens **after** post-hooks. So when a temp relation exists, it is still
present and queryable at post-hook time.

## But whether it exists is conditional

`tmp_relation_exists` is only true when:

```jinja
{% if on_schema_change != 'ignore' or language == 'python' %}
```

…or when the strategy path created one itself — dynamic `insert_overwrite`
always does, static `insert_overwrite` doesn't, `merge` never does on its own.

So the temp relation's existence at post-hook time depends on:

| Setup | Temp relation present? |
| --- | --- |
| `merge`, `on_schema_change: ignore` | **No** |
| `merge`, `on_schema_change: append_new_columns` | Yes |
| `insert_overwrite` dynamic | Yes |
| `insert_overwrite` static | **No**, unless `on_schema_change` forced it |
| Python model | Yes |

Two of those flip based on a config someone will change for unrelated reasons.
`on_schema_change` in particular is set to fix a column problem, by someone not
thinking about your hook.

## Why that makes it a trap

A post-hook reading the temp relation works in dev, works in prod, and then
someone sets `on_schema_change` back to `ignore` to speed up a build — and the
hook fails with a missing table, or worse, silently reads nothing.

The dependency is invisible. Nothing in the hook says "this only works when a
temp table happens to exist", and nothing in the config says "changing this breaks
a hook".

**Don't build on it.** Know the window exists so you can recognise it when
debugging, not so you can use it.

## The legitimate use of knowing this

Debugging. If a post-hook or a manual query behaves oddly mid-run, the temp
relation existing explains extra objects in the dataset. It also explains why
`on_schema_change` changes a model's cost profile — the temp table is a real
materialization of your model's output ([F4](F4-where-hooks-run.md)).

## What people actually want

The usual reason for reaching at the temp relation is "how many rows did this run
write?". Better answers:

**Adapter response in `on-run-end`** — `rows_affected` without a scan, see
[F14](F14-on-run-start-end.md).

**BigQuery job statistics** — `dml_statistics` on the job:

```sql
select
    destination_table.table_id,
    dml_statistics.inserted_row_count,
    dml_statistics.deleted_row_count
from `region-eu`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
where creation_time > timestamp_sub(current_timestamp(), interval 1 hour)
  and destination_table.table_id = 'daily_events';
```

Both give you the number without depending on an implementation detail that
changes with an unrelated config.

## Cleanup is duplicated, not owned

Several strategy paths emit their own `drop table if exists <tmp>` inside the
generated script, *and* the materialization drops it again at the end. That's
belt-and-braces, not a single owner — so don't assume the materialization's drop
is the only one, or that the temp relation survives to any particular point
within the generated SQL.

---

Previous: [F14 · `on-run-start` / `on-run-end`](F14-on-run-start-end.md) ·
Next: [F16 · Hooks and failure semantics](F16-hooks-and-failure.md)
