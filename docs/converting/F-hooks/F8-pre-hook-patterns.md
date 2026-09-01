# F8 · pre-hook patterns worth keeping

> **Part F — Hooks** · Sourcing: `CRAFT`
> **The question:** what is a pre-hook legitimately for?

Short list. Pre-hooks are the less useful of the two, because almost anything you
want to do *before* building a model is better expressed as a dependency.

## The legitimate ones

**Session settings the model needs.**

```sql
{{ config(pre_hook="set @@dataset_project_id = 'analytics-prod'") }}
```

Rare on BigQuery, but real. The setting applies to the session the model runs in.

**Cost guards on an expensive model.**

```sql
{{ config(pre_hook="set @@maximum_bytes_billed = 100000000000") }}
```

Turns a runaway query into a failed one rather than a large bill. Worth it on
models where the input size can change unexpectedly.

**Creating a temporary UDF the model uses.**

```sql
{{ config(pre_hook="{{ create_temp_function_parse_ua() }}") }}
```

Works because the function and the model share a session. A persistent UDF
managed elsewhere is usually better — [C11](../BACKLOG.md#part-c--structural-archetypes) —
but this is a legitimate stopgap.

**Environment-gated setup**, rendering empty elsewhere via
[F3](F3-empty-hook-skipping.md).

That's genuinely most of it.

## What to check before writing one

At pre-hook time:

- the model's output **does not exist yet**
- `{{ this }}` is the *previous* state, or nothing on a first run
- on a `table` model, the old relation has **not** been dropped yet —
  [F5](F5-table-materialization-hooks.md)
- strategy validation has run, but the `copy_partitions` guard has **not** —
  [F4](F4-where-hooks-run.md)

That last one is the sharp edge. A pre-hook can execute, do something
irreversible, and then the model fails on a compiler error. If the hook isn't
safe to run without the model succeeding, it shouldn't be a pre-hook.

## The ones to reject

**"Make sure the upstream table exists."** That's `ref()`. If dbt built it, the
DAG guarantees ordering; if it didn't, declare it a `source` and use freshness
checks. A pre-hook that polls or checks is a dependency written the hard way —
[E2](../E-translation/E2-ordering-by-ref.md).

**"Delete rows before we insert."** Almost always the wrong shape —
[F9](F9-pre-hook-deletes.md) covers why in detail.

**"Truncate the table first."** You want `materialized='table'`, which replaces
wholesale. See [B15](../B-write-patterns/B15-truncate-insert.md).

**"Log that the run started."** Use `on-run-start` if it's per-run, or the
manifest if it's per-model — [F14](F14-on-run-start-end.md). A per-model pre-hook
firing on every model in a 300-model project is 300 log writes.

**"Refresh the upstream data."** That's orchestration. The pipeline that loads
data should finish before dbt starts; expressing it as a pre-hook hides it from
the DAG and from anyone reading the lineage.

## The test

> **Does this need to happen before *this specific model*, and is it not
> expressible as a dependency?**

Both halves have to be true. Most candidate pre-hooks fail the second — they're
dependencies someone wrote as a procedure because that's what the script did.

If it fails either half, the answer is a model, a `ref()`, an `on-run-start`, or
something outside dbt entirely.

---

Previous: [F7 · Ordering within a hook list](F7-hook-ordering.md) ·
Next: [F9 · pre-hook deletes](F9-pre-hook-deletes.md)
