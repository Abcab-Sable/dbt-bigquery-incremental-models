# F16 · Hooks and failure semantics

> **Part F — Hooks** · Sourcing: `SRC` + `CORE✓`
> **The question:** if the model fails, does my post-hook run?

No. And if a hook fails, everything before it stands. There is no rollback on
BigQuery, so partial application is the normal failure state.

## Post-hooks don't run on failure

The materialization is Jinja executed top to bottom:

```jinja
{%- call statement('main') -%}
  {{ build_sql }}
{% endcall %}

{{ run_hooks(post_hooks) }}
```

If the main statement raises, the exception propagates and the materialization
aborts. `run_hooks(post_hooks)` is never reached.

dbt-core catches it at the *node* level, in `safe_run`:

```python
try:
    result = self.compile_and_execute(manifest, ctx)
except Exception as e:
    error = self.handle_exception(e, ctx)
finally:
    exc_str = self._safe_release_connection()
```

That's to record the failure and release the connection — not to resume the
materialization. The post-hook is gone for that run.

**Consequence:** anything that must happen whether or not the model succeeds
cannot be a post-hook. Use `on-run-end`, which does run after failed nodes
([F14](F14-on-run-start-end.md)), or handle it in your orchestrator.

## Pre-hooks run, then the model can still fail

The other direction is worse, because the side effects have already landed.

Pre-hooks are at line 96 of the incremental materialization; the `copy_partitions`
guard is at line 98:

```jinja
{{ run_hooks(pre_hooks) }}

{% if partition_by.copy_partitions is true and strategy not in ['insert_overwrite', 'microbatch'] %}
    {% do exceptions.raise_compiler_error(wrong_strategy_msg) %}
```

So a misconfigured model can run your pre-hook and *then* fail on a compiler
error. Anything irreversible in a pre-hook needs to be safe in that world —
[F9](F9-pre-hook-deletes.md) is the case where it isn't.

## Within a hook list, failure stops the rest

Each hook is its own `statement` call. Hook 2 failing means hook 3 doesn't run,
and hook 1's effect stands. No rollback.

This is a reason to prefer one macro over a list of hooks
([F7](F7-hook-ordering.md)): a single statement either applies or doesn't, with
no partial state to reason about.

## `transaction: true` doesn't help

The name suggests atomicity. On BigQuery there is no wrapping transaction — dbt
never issues `BEGIN` for these materializations, and `run_hooks` is called only at
the `True` default ([F6](F6-transaction-filter.md)). The flag selects which phase
a hook belongs to; it does not make anything atomic.

## Microbatch: the boundary batches

For microbatch models, dbt-core clears hooks on non-boundary batches:

```python
# Only run pre_hook(s) for first batch
if batch_idx != 0:
    node_copy.config.pre_hook = []

# Only run post_hook(s) for last batch
if batch_idx != len(batches) - 1:
    node_copy.config.post_hook = []
```

Which raises a question worth answering explicitly: **if batch 200 of 400 fails,
does the post-hook run?**

It's attached to the last batch. If the run doesn't reach the last batch, the
post-hook doesn't fire. If batches are retried and the last one eventually
succeeds, it fires then.

So a post-hook on a microbatch model signals "the final batch completed", not
"all batches succeeded". Don't use one as a completion signal for a backfill.

## Designing for it

- Anything that must always happen ⇒ `on-run-end`, or the orchestrator
- Anything irreversible ⇒ not a pre-hook
- Anything with several steps ⇒ one macro, not a list
- Anything used as a completion signal ⇒ not a hook at all

The general rule: **hooks are best-effort side effects attached to a build.**
Treat them as guarantees and you'll be surprised at 3am.

---

Previous: [F15 · Hooks and the temp relation](F15-hooks-and-temp-relation.md) ·
Next: [H1 · What "correct" means](../H-verification/H1-what-correct-means.md)
