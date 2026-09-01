# F4 · Exactly where hooks run

> **Part F — Hooks** · Sourcing: `SRC`
> **The question:** at the moment my hook fires, what exists and what doesn't?

Hook guidance is usually written as "before" and "after". That isn't precise
enough to convert a script safely — "after" the main statement is not the same as
after grants, or after the temp table is dropped. Here is the actual position.

## The two call sites

The BigQuery incremental materialization calls `run_hooks` exactly twice.
Everything else in this page follows from where those two calls sit.

```jinja
{% set strategy = dbt_bigquery_validate_get_incremental_strategy(config) %}   ← validation
{% set raw_partition_by = config.get('partition_by', none) %}                 ← config parsed
{% set on_schema_change = incremental_validate_on_schema_change(...) %}
{% set grant_config = config.get('grants') %}

{{ run_hooks(pre_hooks) }}                          ←← PRE-HOOKS

{% if partition_by.copy_partitions ... %}           ← compiler error possible
{% elif existing_relation is none %} ...            ← the branch dispatch
{% else %} ... {% endif %}                          ← main statement runs here

{{ run_hooks(post_hooks) }}                         ←← POST-HOOKS

{% set target_relation = this.incorporate(type='table') %}
{% do apply_grants(target_relation, grant_config, should_revoke) %}
{% do persist_docs(target_relation, model) %}
{%- if tmp_relation_exists -%}
  {{ adapter.drop_relation(tmp_relation) }}
{%- endif -%}
```

## What that means for pre-hooks

**Already happened:** strategy validated, `partition_by` parsed,
`on_schema_change` validated, grant config read.

**Not yet happened:** any table creation, the temp relation, the main statement.

So a pre-hook runs against the **previous** state of the world. `{{ this }}` is
last run's table, or nothing at all on a first run. If your hook assumes the
model's output exists, it will not.

### The sharp edge

Pre-hooks run **before** the `copy_partitions` guard, not after.

An invalid `incremental_strategy` fails at validation, above the hook — nothing
runs. But `copy_partitions` combined with `merge` raises its compiler error at
the branch *below* the hook. Your pre-hook will have already executed, and its
side effects will already have happened, before the model fails.

If your pre-hook does something irreversible, that's the case to know about.

## What that means for post-hooks

This is the part people get wrong. Post-hooks run **after** the main statement,
but **before** three things:

| Runs after post-hooks | So a post-hook… |
| --- | --- |
| `apply_grants` | cannot assume grants are applied — see [F11](../F-hooks/F11-grants-vs-post-hook.md) |
| `persist_docs` | cannot assume descriptions are on the relation |
| `drop_relation(tmp_relation)` | **can still see the temp table**, if one existed |

That last row is occasionally useful and frequently surprising. When
`on_schema_change` is not `ignore`, or on the dynamic `insert_overwrite` path, a
temp relation exists at post-hook time and hasn't been dropped yet.

Relying on it is fragile — whether a temp relation exists at all depends on your
`on_schema_change` setting and your strategy, both of which someone will change.
Know it's there; don't build on it.

The grants ordering matters more. A post-hook issuing `GRANT` runs *before*
dbt's own `apply_grants`, so the two race rather than compose. If `should_revoke`
is true, dbt may revoke what your hook just granted. Use the `grants` config
instead.

## Two calls, not four

The default (non-BigQuery) incremental materialization calls `run_hooks` four
times:

```jinja
{{ run_hooks(pre_hooks, inside_transaction=False) }}
{{ run_hooks(pre_hooks, inside_transaction=True) }}
...
{{ run_hooks(post_hooks, inside_transaction=True) }}
{{ run_hooks(post_hooks, inside_transaction=False) }}
```

**BigQuery calls it twice, both at the default `inside_transaction=True`.**

`run_hooks` filters its input:

```jinja
{% for hook in hooks | selectattr('transaction', 'equalto', inside_transaction) %}
```

So on BigQuery, a hook configured `transaction: false` is filtered out and
**silently never runs**. No error, no log line.

The good news, from dbt-core: `Hook.transaction` defaults to `True`. A
plain-string hook — which is what almost everyone writes — is unaffected. Only an
explicit `transaction: false` disappears.

## Two more mechanics worth knowing

**Empty hooks are skipped.** `run_hooks` only emits a statement when
`(rendered | length) > 0`. A hook whose Jinja renders to nothing is a no-op, not
an error. That's what makes conditional hooks viable:

```sql
post_hook="{% if target.name == 'prod' %} grant select on {{ this }} to 'group:analysts' {% endif %}"
```

In dev that renders empty and is skipped entirely.

**Hooks are rendered at hook time.** `run_hooks` calls `render(hook.get('sql'))`,
so Jinja inside a hook is evaluated when the hook runs, in the model's context.
`{{ this }}` resolves correctly.

## Microbatch: once per model, not per batch

If the model uses `microbatch`, dbt-core clears hooks on all but the boundary
batches:

```python
# Only run pre_hook(s) for first batch
if batch_idx != 0:
    node_copy.config.pre_hook = []

# Only run post_hook(s) for last batch
if batch_idx != len(batches) - 1:
    node_copy.config.post_hook = []
```

A 400-batch backfill runs each hook **once**. If you converted a per-run script
step into a post-hook expecting it to fire per batch, it won't.

## The summary you can act on

- Pre-hook: previous state only. Model output doesn't exist yet.
- Pre-hook: can run and *then* have the model fail on the `copy_partitions` guard.
- Post-hook: model output exists; grants and docs do **not** yet.
- Post-hook: the temp relation may still exist. Don't depend on it.
- `transaction: false` never fires on BigQuery. The default is `True`, so this is
  narrow.
- Empty-rendering hooks are skipped, which makes conditionals work.
- Microbatch runs hooks once per model.

---

Previous: [F17 · When a hook is the wrong answer](F17-when-a-hook-is-wrong.md) ·
Next in wave 1: [H2 · Row-count parity](../H-verification/H2-row-count-parity.md)
