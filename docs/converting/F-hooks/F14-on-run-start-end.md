# F14 · `on-run-start` / `on-run-end` vs per-model hooks

> **Part F — Hooks** · Sourcing: `CORE✓`
> **The question:** my script logs at start and end. Per model, or per run?

Per run, almost always. A script had one start and one end; a per-model hook
gives you one per model, which is a different thing wearing the same name.

## The two scopes

dbt-core defines them as run-level hook types:

```python
class RunHookType(StrEnum):
    Start = "on-run-start"
    End = "on-run-end"
```

Configured in `dbt_project.yml`, not on a model:

```yaml
on-run-start:
  - "{{ log_run_start() }}"

on-run-end:
  - "{{ log_run_end(results) }}"
```

They fire **once per invocation**, regardless of how many models run.

| | Fires | Use for |
| --- | --- | --- |
| `on-run-start` | once, before any model | run-level setup, logging the start |
| `on-run-end` | once, after all models | run-level summary, notification triggers |
| `pre_hook` | once per model | things about *that model* |
| `post_hook` | once per model | things about *that model* |

## Mapping your script

A script's "log the start / log the end" is run-scoped. One script, one run, one
of each. Converting that to a per-model hook multiplies it by your model count and
changes its meaning.

```yaml
# right — one row per dbt invocation
on-run-end:
  - "insert into ops.run_log (run_id, finished_at) values ('{{ invocation_id }}', current_timestamp())"
```

`invocation_id` gives you a stable identifier for the run, which is what makes the
row joinable to anything else you record.

## `results` is available in `on-run-end`

The distinguishing feature: `on-run-end` gets a `results` context containing every
node's outcome. That's what makes a genuine run summary possible in one statement
rather than N:

```sql
{% macro log_run_end(results) %}
  {% if execute and results %}
    insert into ops.run_log (invocation_id, node, status, rows_affected, finished_at)
    values
    {%- for res in results %}
      (
        '{{ invocation_id }}',
        '{{ res.node.unique_id }}',
        '{{ res.status }}',
        {{ res.adapter_response.get('rows_affected', 'null') }},
        current_timestamp()
      ){{ ',' if not loop.last }}
    {%- endfor %}
  {% endif %}
{% endmacro %}
```

One statement, every model covered, no per-model scan. This is strictly better
than the per-model audit hook in [F13](F13-post-hook-audit-rows.md), and it's the
migration target for it.

The `{% if execute %}` guard matters — the macro is evaluated during parsing too,
when `results` doesn't exist.

## `on-run-end` runs even when models fail

That's the point of it — a run summary that only appears on success isn't a
summary. It means the macro must tolerate failed nodes:

- `res.status` may be `error` or `skipped`
- `res.adapter_response` may be empty
- `res.node` still exists

Use `.get()` with defaults rather than assuming, as above.

## The comparison to a script's exit handler

A script's `END` block runs on the script's terms. `on-run-end` runs on dbt's,
and the mapping is close but not identical:

| Script | dbt |
| --- | --- |
| Runs after the last statement | Runs after the last node |
| Sees whether the script failed | Sees per-node status via `results` |
| Runs once | Runs once |
| Can abort the script | Cannot un-build models |

That last row matters if the script's end block did cleanup on failure. dbt has
already built whatever succeeded; `on-run-end` can record that, not undo it.

## Don't use it for orchestration

`on-run-end` firing a notification is reasonable. `on-run-end` kicking off the
next pipeline stage is your scheduler's job, and putting it here hides a
dependency from everyone who reads the project —
[K4](../K-antipatterns/K4-run-operation-as-scheduler.md).

---

Previous: [F13 · post-hook: audit rows](F13-post-hook-audit-rows.md) ·
Next: [F15 · Hooks and the temp relation](F15-hooks-and-temp-relation.md)
