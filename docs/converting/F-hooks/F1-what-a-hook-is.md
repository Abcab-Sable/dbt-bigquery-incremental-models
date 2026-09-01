# F1 · What a hook actually is

> **Part F — Hooks** · Sourcing: `SRC`
> **The question:** what does dbt do with the string I put in `post_hook`?

A hook is a SQL string that a materialization executes as a separate statement.
That's the whole mechanism, and it's smaller than most people assume.

## The macro, in full

From `dbt-adapters`, `macros/materializations/hooks.sql`:

```jinja
{% macro run_hooks(hooks, inside_transaction=True) %}
  {% for hook in hooks | selectattr('transaction', 'equalto', inside_transaction) %}
    {% if not inside_transaction and loop.first %}
      {% call statement(auto_begin=inside_transaction) %}
        commit;
      {% endcall %}
    {% endif %}
    {% set rendered = render(hook.get('sql')) | trim %}
    {% if (rendered | length) > 0 %}
      {% call statement(auto_begin=inside_transaction) %}
        {{ rendered }}
      {% endcall %}
    {% endif %}
  {% endfor %}
{% endmacro %}
```

Twelve lines. Four behaviours worth knowing come out of it.

## 1. Hooks are filtered before they run

```jinja
{% for hook in hooks | selectattr('transaction', 'equalto', inside_transaction) %}
```

Only hooks whose `transaction` attribute equals the phase being run are executed.
Everything else is skipped without comment. On BigQuery this has a specific
consequence — [F6](F6-transaction-filter.md).

## 2. Each hook is its own statement

`{% call statement(...) %}` per hook. They are not concatenated, not wrapped, and
not run as a script. A hook containing two statements separated by a semicolon
may work on BigQuery — which accepts multi-statement scripts — but each *hook* is
one execution.

## 3. Hooks are rendered at run time

`render(hook.get('sql'))` — the Jinja inside a hook is evaluated when the hook
runs, in the model's context. `{{ this }}`, `{{ target }}` and `var()` all
resolve. Details in [F2](F2-hook-rendering.md).

## 4. Empty hooks vanish

`{% if (rendered | length) > 0 %}` — a hook rendering to nothing produces no
statement at all. This is what makes conditional hooks work rather than
erroring. See [F3](F3-empty-hook-skipping.md).

## The shape of a hook config

Two forms. A bare string:

```sql
{{ config(post_hook="grant select on {{ this }} to 'group:analysts'") }}
```

Or a dict, when you need to set `transaction`:

```sql
{{ config(post_hook={"sql": "...", "transaction": false}) }}
```

dbt-core's `Hook` dataclass defines the defaults:

```python
@dataclass
class Hook(dbtClassMixin):
    sql: str
    transaction: bool = True
    index: Optional[int] = None
```

So a bare string becomes `transaction: True`, which is the value BigQuery
materializations actually use. Ordinary hooks work; see
[F6](F6-transaction-filter.md) for when they don't.

## Lists run in order

```sql
{{ config(post_hook=["statement one", "statement two"]) }}
```

dbt-core assigns `index` from `enumerate(hooks)` at parse time, and the macro
iterates the list. Order is preserved — [F7](F7-hook-ordering.md).

## What a hook is not

- **Not part of the model's definition.** It doesn't affect the built relation's
  contents unless it separately modifies them, which is usually a mistake.
- **Not transactional with the model** on BigQuery. Each hook is its own
  statement; there's no rollback if a later one fails.
- **Not skipped on failure of a later hook.** Earlier statements already ran.
- **Not a place for DML against other tables.** That's the anti-pattern —
  [F17](F17-when-a-hook-is-wrong.md).

---

Next: [F2 · Hook rendering](F2-hook-rendering.md) ·
Back to [the backlog](../BACKLOG.md)
