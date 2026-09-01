# F17 · When a hook is the wrong answer

> **Part F — Hooks** · Sourcing: `CRAFT`
> **The question:** my script has statements left over. Do they become hooks?

Usually not. Hooks are how a conversion quietly fails: the leftover statements go
into a `post_hook`, the model runs green, and you've rebuilt the script inside a
model that now pretends to be declarative.

## The test

For each leftover statement, ask:

> **Is this part of what the table *is*, or something that happens *around* it?**

Part of what it is ⇒ **another model**, or logic inside this one.
Something around it ⇒ a hook is a candidate.

That single question resolves most cases. The rest of this page is what to do
when it doesn't.

## The tell

If your hook contains `INSERT`, `MERGE`, `UPDATE`, `DELETE`, or
`CREATE TABLE ... AS SELECT` **against a table that isn't this model**, you are
building a second model inside the first one.

That second table now has:

- no entry in the DAG, so nothing can `ref()` it
- no lineage, so nobody can see where it came from
- no tests, no docs, no `--select` targeting
- a build order determined by which model happens to own the hook

That's every property dbt exists to provide, given up to avoid creating a file.

## Legitimate uses

Hooks are right for things that are genuinely *about* the relation but not *part
of* it:

- **Table metadata** — labels, description, expiration ([D7](../D-data-movement/D7-table-options.md))
- **Audit rows** — recording that this model ran, with row counts ([F13](../F-hooks/F13-post-hook-audit-rows.md))
- **Session settings** a specific model needs
- **Environment-conditional operations** that render empty elsewhere

Short, idempotent, and about *this* relation. That's the shape.

## Where the leftovers actually go

| Your script's leftover | Not a hook | Because |
| --- | --- | --- |
| Builds an intermediate table | **A model** | It's a table. Give it a name and lineage. |
| Writes a second output table | **A model** | Two outputs, two models ([C4](../C-structural/C4-fan-out.md)) |
| `DELETE`s old rows for retention | **A model, or a scheduled operation** | It's a policy, not part of the definition |
| Checks a condition and fails | **A dbt test** | That's precisely what tests are ([D12](../D-data-movement/D12-assert-gates.md)) |
| Exports to GCS | **Orchestration** | Not a transformation ([D2](../D-data-movement/D2-export-data.md)) |
| Sends a notification | **Orchestration** | Your scheduler already does this |
| Grants access | **The `grants` config** | A hook races `apply_grants` — [F11](../F-hooks/F11-grants-vs-post-hook.md) |
| Creates a UDF | **A macro or managed DDL** | ([C11](../C-structural/C11-temp-functions.md)) |
| One-off maintenance | **`run-operation`** | Not per-model work |

## Four reasons hooks are worse than they look

Specific to BigQuery, all verified in source, all covered in
[F4](F4-where-hooks-run.md):

**Post-hooks run before `apply_grants`.** A hook that grants access races dbt's
own grant handling instead of composing with it.

**Pre-hooks can run and then the model fails.** The `copy_partitions` guard
raises *after* pre-hooks have executed. Irreversible side effects will already
have happened.

**`transaction: false` never fires.** Every BigQuery materialization calls
`run_hooks` only at `inside_transaction=True`, so that hook is filtered out
silently. The default is `True`, so this is narrow — but it's invisible when it
bites.

**Microbatch hooks fire once per model, not per batch.** A 400-batch backfill
runs your hook once. If the leftover statement was per-run logic, its semantics
just changed.

## The honest cases for keeping one

Sometimes a hook really is right, and the reason is usually organisational rather
than technical:

- The side effect must be atomic with this model and you don't control the
  orchestrator
- It's genuinely tiny, idempotent, and about this relation
- It's a stopgap during migration and there's a ticket to remove it

The last is fine as long as the ticket is real. Write the reason in a comment
next to the hook:

```sql
{{ config(
    materialized='incremental',
    unique_key='order_id',
    -- TEMPORARY (DATA-2291): mirrors the legacy audit row the old script wrote.
    -- Remove once the audit pipeline reads from the manifest instead.
    post_hook="insert into ops.model_audit (model, run_at) values ('orders', current_timestamp())"
) }}
```

A hook with a reason is a decision. A hook without one is a leftover, and in six
months nobody will know which.

## The rule of thumb

**If you can't explain why it isn't a model, it should be a model.**

Conversions that go wrong nearly always go wrong here — not by picking the wrong
strategy, but by moving the script's imperative tail into hooks and calling the
job done. That's [K2](../K-antipatterns/K2-hooks-as-escape-hatch.md), and it's the most
common bad outcome in the whole track.

---

Previous: [B14 · When the range can legitimately empty](../B-write-patterns/B14-when-the-range-can-empty.md) ·
Next: [F4 · Exactly where hooks run](F4-where-hooks-run.md)
