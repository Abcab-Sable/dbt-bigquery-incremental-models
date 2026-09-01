# F5 · Where hooks run in the `table` materialization

> **Part F — Hooks** · Sourcing: `SRC`
> **The question:** is hook position the same for a `table` model as an incremental one?

Structurally yes, with one difference that matters if your pre-hook cares about
the old table.

## The positions

From `dbt-bigquery`, `materializations/table.sql`:

```
line 12   run_hooks(pre_hooks)
line 20   drop_relation(old_relation)     ← conditional path
line 29   drop_relation(old_relation)     ← conditional path
line 33   statement('main')               ← the CREATE TABLE AS
line 37   run_hooks(post_hooks)
line 40   apply_grants(...)
line 42   persist_docs(...)
```

Compare to [incremental](F4-where-hooks-run.md): pre-hooks first, main statement,
post-hooks, then grants and docs. The **post-hook side is identical** — still
before `apply_grants` and `persist_docs`, with the same
[grants ordering trap](F11-grants-vs-post-hook.md).

## The difference: pre-hooks run before the drop

In the `table` materialization, `drop_relation(old_relation)` happens **after**
`run_hooks(pre_hooks)`.

So at pre-hook time the **old table still exists**. You can read it, count it,
snapshot it:

```sql
{{ config(
    materialized='table',
    pre_hook="create or replace table {{ this }}_prev as select * from {{ this }}"
) }}
```

That works on a rebuild — and fails on the very first run, when `{{ this }}`
doesn't exist yet. Guard it:

```sql
pre_hook="{% if adapter.get_relation(this.database, this.schema, this.identifier) %} ... {% endif %}"
```

Or accept that it only makes sense after the first build and let
[F3](F3-empty-hook-skipping.md) skip it.

The incremental materialization has no unconditional drop, so this distinction is
specific to `table` — and to the view-replacement path.

## Both are two calls, not four

Like the incremental materialization, `table.sql` calls `run_hooks` exactly twice,
both at the default `inside_transaction=True`. The four-call
inside/outside-transaction pattern is a default-materialization thing that
BigQuery doesn't do. Consequences in [F6](F6-transaction-filter.md).

## The other BigQuery materializations

Same two-call shape throughout:

| Materialization | pre | post |
| --- | --- | --- |
| `incremental` | line 96 | line 165 |
| `table` | line 12 | line 37 |
| `copy` | line 4 | line 28 |
| view replace | line 23 | line 40 |

If you're converting a script whose model might change materialization later —
`view` to `table`, or `table` to `incremental` — the post-hook contract stays the
same. The pre-hook's view of the world does not.

## Practical consequence for conversions

A pre-hook that reads `{{ this }}` behaves differently across materializations:

| Materialization | At pre-hook time, `{{ this }}` is |
| --- | --- |
| `table` | the **old** table, not yet dropped |
| `incremental` | the **current** table, about to be merged into |
| first run of either | **nonexistent** |

If your script had a "back up the old version first" step and you're porting it
to a pre-hook, that only works on `table`. On incremental it backs up a table
that's about to be modified in place, which may be what you want or may not.

Better answer for that particular need: a snapshot, or a proper backup process
outside dbt. See [F17](F17-when-a-hook-is-wrong.md).

---

Previous: [F3 · Empty-hook skipping](F3-empty-hook-skipping.md) ·
Next: [F6 · The `transaction` filter](F6-transaction-filter.md)
