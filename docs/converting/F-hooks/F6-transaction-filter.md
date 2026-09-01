# F6 · The `transaction` filter, and BigQuery

> **Part F — Hooks** · Sourcing: `SRC` + `CORE✓`
> **The question:** does `transaction: false` do anything on BigQuery?

It makes the hook **never run**. Silently. But the default is `True`, so this only
bites if you set it explicitly — which makes it a narrow trap rather than a broad
one.

## The filter

`run_hooks` selects only hooks matching the phase it was invoked for:

```jinja
{% for hook in hooks | selectattr('transaction', 'equalto', inside_transaction) %}
```

Anything not matching is skipped. No error, no log line.

## BigQuery calls it once per phase

The default (non-BigQuery) incremental materialization calls `run_hooks` **four**
times:

```jinja
{{ run_hooks(pre_hooks, inside_transaction=False) }}
-- BEGIN happens here
{{ run_hooks(pre_hooks, inside_transaction=True) }}
...
{{ run_hooks(post_hooks, inside_transaction=True) }}
-- COMMIT happens here
{{ run_hooks(post_hooks, inside_transaction=False) }}
```

Every BigQuery materialization calls it **twice**, both at the default:

```jinja
{{ run_hooks(pre_hooks) }}     -- inside_transaction=True
{{ run_hooks(post_hooks) }}    -- inside_transaction=True
```

So on BigQuery there is no `inside_transaction=False` pass. A hook carrying
`transaction: false` matches nothing and is dropped.

## The default saves you

From dbt-core, `artifacts/resources/v1/config.py`:

```python
@dataclass
class Hook(dbtClassMixin):
    sql: str
    transaction: bool = True
    index: Optional[int] = None
```

`transaction` defaults to `True`. A plain-string hook — what almost everyone
writes — gets `True` and runs normally.

**Only this form disappears:**

```sql
{{ config(post_hook={"sql": "...", "transaction": false}) }}
```

## Where it comes from in a conversion

You'd write `transaction: false` for one of two reasons, both inherited from
warehouses that aren't BigQuery:

- Copied from a Snowflake or Postgres project, where it means "run outside the
  transaction"
- Copied from a Stack Overflow answer or blog post written for those warehouses

Neither reason applies on BigQuery, where dbt isn't wrapping your model in a
transaction to begin with. The config is inert here — and worse than inert,
because it removes the hook.

## Check for it

```bash
grep -rn "transaction" models/ | grep -i "false"
```

Any hit is a hook that isn't running. Convert it to a plain string, or drop the
`transaction` key:

```sql
-- before: never runs on BigQuery
post_hook={"sql": "alter table {{ this }} set options(...)", "transaction": false}

-- after
post_hook="alter table {{ this }} set options(...)"
```

## Corroboration from dbt-core

dbt-core has a helper that mirrors the macro's filter for telemetry:

```python
def _hook_count(hooks: Any, inside_transaction: bool) -> int:
    """How many hooks `run_hooks` actually executes in this transaction phase.

    Mirrors the macro's own `selectattr('transaction', ...)` filter: a ...
    """
    return sum(1 for hook in hooks if hook["transaction"] == inside_transaction)
```

Same filter, independently implemented — confirming the behaviour is intended
rather than incidental.

## What it doesn't mean

`transaction: true` does **not** mean your hook is atomic with the model on
BigQuery. There is no wrapping transaction; the name is inherited from warehouses
that have one. Each hook is a separate statement, and a failure part-way through
leaves earlier ones applied — [F16](F16-hooks-and-failure.md).

---

Previous: [F5 · Where hooks run in the `table` materialization](F5-table-materialization-hooks.md) ·
Next: [F7 · Ordering within a hook list](F7-hook-ordering.md)
