# F3 · Empty-hook skipping

> **Part F — Hooks** · Sourcing: `SRC`
> **The question:** what happens if my conditional hook renders to nothing?

Nothing happens, which is exactly what you want. The statement is never issued.

## The guard

```jinja
{% set rendered = render(hook.get('sql')) | trim %}
{% if (rendered | length) > 0 %}
  {% call statement(auto_begin=inside_transaction) %}
    {{ rendered }}
  {% endcall %}
{% endif %}
```

Note the `| trim` — whitespace-only counts as empty. A hook that renders to
newlines and spaces is skipped too, which matters because Jinja block tags leave
whitespace behind.

## Why this is the enabling behaviour

Without it, every conditional hook would need to render *something* valid, and
you'd see workarounds like `select 1` as a no-op statement. Instead:

```sql
{{ config(
    post_hook="{% if target.name == 'prod' %} grant select on {{ this }} to 'group:analysts' {% endif %}"
) }}
```

In dev, the `if` is false, the hook renders to whitespace, `trim` reduces it to
empty, and no statement runs. No error, no wasted round-trip.

This is the idiomatic pattern for environment-specific hooks, and it works
because of these three lines.

## Useful applications

**Environment gating** — as above.

**Config-driven hooks** — render nothing when a variable isn't set:

```sql
post_hook="{% if var('apply_labels', false) %} alter table {{ this }} set options(labels=[('team','analytics')]) {% endif %}"
```

**First-run-only work** — combined with `is_incremental()`:

```sql
post_hook="{% if not is_incremental() %} ... {% endif %}"
```

**Disabling without deleting** — wrap in `{% if false %}` to switch a hook off
while keeping it visible in the code and in review.

## The failure mode it causes

The same behaviour that makes conditionals clean makes typos silent.

```sql
post_hook="{% if target.name == 'production' %} ... {% endif %}"
```

If your target is actually named `prod`, this renders empty **every time** and is
skipped **every time**. No error. The grant never happens, and you find out when
someone can't read the table.

There's no "hook did nothing" log line to catch it, so the mistake persists.

## Checking a hook actually fires

Don't assume — verify:

```bash
dbt run --select my_model --target prod
```

Then look for the hook's statement in the run log, or confirm its effect directly:

```sql
select * from `project.dataset.INFORMATION_SCHEMA.OBJECT_PRIVILEGES`
where object_name = 'my_model';
```

For anything where silent non-execution matters — grants, retention, audit rows —
check the effect rather than trusting the config. A hook that renders empty looks
identical to a hook that isn't there.

---

Previous: [F2 · Hook rendering](F2-hook-rendering.md) ·
Next: [F5 · Where hooks run in the `table` materialization](F5-table-materialization-hooks.md)
