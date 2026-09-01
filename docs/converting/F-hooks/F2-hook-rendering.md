# F2 · Hook rendering: when the Jinja is evaluated

> **Part F — Hooks** · Sourcing: `SRC`
> **The question:** can I use `{{ this }}` and `ref()` in a hook?

`{{ this }}` yes. `ref()` — technically yes, and it's a trap. The distinction
comes from *when* the hook's Jinja is evaluated.

## The rendering call

`run_hooks` renders each hook immediately before executing it:

```jinja
{% set rendered = render(hook.get('sql')) | trim %}
```

So the Jinja in a hook is evaluated at **hook execution time**, inside the
running materialization, with the model's context available. That's why
`{{ this }}` works:

```sql
{{ config(post_hook="alter table {{ this }} set options(description='Daily events')") }}
```

At the point that runs, `{{ this }}` is the model's relation, fully qualified.

## Quoting is the practical problem

A hook is a string inside a config block, containing Jinja that must survive to
be rendered later. Quote nesting gets awkward fast.

Prefer single quotes outside, double inside — or the other way — but don't mix in
the same string:

```sql
-- fine
{{ config(post_hook="grant select on {{ this }} to 'group:analysts'") }}

-- fine, multi-line
{{ config(
    post_hook="""
      alter table {{ this }}
      set options(description = 'Daily event counts by user')
    """
) }}
```

When a hook grows past a line or two, put it in a macro and call that:

```sql
{{ config(post_hook="{{ apply_table_labels(this) }}") }}
```

The macro is testable, reusable, and doesn't fight the quoting. This is the right
default for anything non-trivial.

## `ref()` in a hook creates no edge

You can write `ref()` in a hook and it will resolve. But the dependency graph is
built from `depends_on_nodes`, which is populated at **parse** time from the
model body — and a hook referencing another model does not reliably register
there in the way a body reference does.

The practical rule: **do not rely on a hook's `ref()` to order anything.** If your
hook reads another model, dbt may not guarantee that model was built first. Put
the reference in the model body if the ordering matters, or restructure so the
hook doesn't need it. See [E5](../E-translation/E5-finding-hardcoded-names.md) —
hooks are one of the places hardcoded names hide.

## `target` makes hooks environment-aware

The most useful thing you can do with hook rendering:

```sql
{{ config(
    post_hook="{% if target.name == 'prod' %} grant select on {{ this }} to 'group:analysts' {% endif %}"
) }}
```

In dev this renders empty, and [F3](F3-empty-hook-skipping.md) means it's skipped
entirely rather than erroring. That's the idiomatic way to make a hook production-
only.

## It renders every run

There's no caching. A hook doing something expensive in Jinja — a loop over a
large list, a `run_query` to build a string — pays that cost on every model run.

Hooks that call `run_query` at render time are a particular smell: you've added a
warehouse round-trip to every build, invisibly. If you need data to decide what a
hook does, that's usually a sign the work belongs in a model.

---

Previous: [F1 · What a hook actually is](F1-what-a-hook-is.md) ·
Next: [F3 · Empty-hook skipping](F3-empty-hook-skipping.md)
