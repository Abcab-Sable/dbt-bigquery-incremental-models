# F7 · Ordering within a hook list

> **Part F — Hooks** · Sourcing: `SRC`
> **The question:** if I give a list of hooks, do they run in order?

Yes. List order is preserved, indexed at parse time, and iterated in sequence.

## The mechanism

dbt-core assigns an index as it parses the list (`parser/hooks.py`):

```python
for index, hook in enumerate(hooks):
    ...
    index=index,
```

And the macro iterates the list directly:

```jinja
{% for hook in hooks | selectattr('transaction', 'equalto', inside_transaction) %}
```

`selectattr` preserves order, so filtered hooks keep their relative sequence.

```sql
{{ config(post_hook=[
    "alter table {{ this }} set options(description='Daily events')",
    "alter table {{ this }} set options(labels=[('team','analytics')])"
]) }}
```

Description first, labels second, every run.

## Where ordering is not guaranteed

**Between project-level and model-level hooks.** Hooks defined in
`dbt_project.yml` and hooks on the model both apply. Don't build a sequence that
depends on how those interleave.

**Between hooks and the things after them.** Post-hooks all run before
`apply_grants` and `persist_docs`, as a block — [F4](F4-where-hooks-run.md). You
cannot interleave a hook between them.

**Across models.** Model A's post-hook and model B's pre-hook have no ordering
relationship beyond the DAG's, and if the models run in parallel there is none at
all.

## The dependency smell

If your hooks *must* run in a specific order because hook 2 depends on hook 1's
effect, ask what you're actually building. A sequence of dependent statements
attached to a model is the script you were converting away from —
[E1](../E-translation/E1-one-statement-per-model.md).

Two or three independent statements (description, labels, expiration) is fine.
A five-step sequence where each depends on the last is
[K2](../BACKLOG.md#part-k--anti-patterns).

## Prefer a macro for anything ordered

If the sequence genuinely matters, put it in one macro rather than several hooks:

```sql
{{ config(post_hook="{{ finalise_table(this) }}") }}
```

```sql
{% macro finalise_table(relation) %}
  alter table {{ relation }} set options(
      description = 'Daily event counts',
      labels = [('team', 'analytics')]
  )
{% endmacro %}
```

One hook, one statement, order explicit in the macro body. Easier to read, easier
to test, and it can't be reordered by someone editing a list.

## Failure stops the rest

If hook 2 of 3 fails, hook 3 does not run — and hook 1's effect stands. There's no
rollback across hooks on BigQuery. See [F16](F16-hooks-and-failure.md).

This is another argument for fewer, larger hooks: a single statement either
succeeds or doesn't, whereas a list has partial-failure states.

---

Previous: [F6 · The `transaction` filter](F6-transaction-filter.md) ·
Next: [F8 · pre-hook patterns worth keeping](F8-pre-hook-patterns.md)
