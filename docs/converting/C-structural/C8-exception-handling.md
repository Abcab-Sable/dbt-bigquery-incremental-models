# C8 · `BEGIN ... EXCEPTION WHEN ERROR`

> **Part C — Structural archetypes** · Sourcing: `CRAFT`
> **The question:** my script catches its own errors. What replaces that?

dbt's failure model, mostly. A model either succeeds or fails; there's no
try/catch inside one. What the handler was doing decides where it goes.

## Work out what the handler is for

```sql
BEGIN
    INSERT INTO analytics.daily_events SELECT ... ;
EXCEPTION WHEN ERROR THEN
    INSERT INTO ops.errors VALUES (@@error.message, CURRENT_TIMESTAMP());
END;
```

Three common purposes, three different answers.

**Logging the failure.** dbt already does this — the run result records the
error, the message, and the node. If you need it in a table, `on-run-end` has
access to `results` including failed nodes
([F14](../F-hooks/F14-on-run-start-end.md)). One statement per run, and it
covers every model rather than the one you remembered to wrap.

**Continuing despite the failure.** dbt does this by default: a failed model
doesn't stop the run, it stops that model's *descendants*. Other branches
continue. So the handler's purpose is already the behaviour.

If you specifically want downstream models to build anyway, that's a
`severity: warn` test rather than an error, or a design change.

**Cleaning up partial state.** This one doesn't convert, and it usually doesn't
need to — see below.

## Cleanup handlers are usually unnecessary now

A script needed cleanup because its statements weren't atomic: it deleted, then
inserted, and a failure between the two left a hole.

The dbt equivalents don't have that gap:

| Script | dbt | Failure leaves |
| --- | --- | --- |
| `DELETE` then `INSERT` | `insert_overwrite` | one `MERGE` — no partial state |
| `TRUNCATE` then `INSERT` | `materialized='table'` | the old table intact |
| `MERGE` | `merge` | one statement |

So the handler protected against a failure mode the conversion removes. Confirm
that's true for your case, then delete it.

The exception is a **pre-hook** that modifies data — which reintroduces exactly
the gap, with no handler. That's [F9](../F-hooks/F9-pre-hook-deletes.md), and
it's a reason not to write one.

## What genuinely has no equivalent

**Retrying inside the script.** dbt retries at the invocation level
(`dbt retry`), not inside a model. Transient-failure handling belongs to your
orchestrator.

**Suppressing an error and continuing with partial data.** dbt won't build a
model from a query that errored. If your script caught an error and inserted what
it had, that behaviour is gone — and it's worth asking whether it was ever
desirable, since it produces silently incomplete data.

**Custom error messages.** `exceptions.raise_compiler_error()` exists for
compile-time validation in macros, but not for wrapping a model's runtime
failure.

## The improvement worth naming

The script's handler ran only where someone wrote one. dbt's failure handling is
uniform: every node, every run, same reporting, same artefacts, same exit code.

A conversion typically *increases* error visibility, because the handler was
usually swallowing the error into a table nobody read. Check whether
`ops.errors` has been written to recently — if it has and nobody noticed, that's
a finding for [A9](../A-assess/A9-correctness-baseline.md):

```sql
select date(logged_at) as d, count(*) from ops.errors
where logged_at > timestamp_sub(current_timestamp(), interval 90 day)
group by 1 order by 1 desc;
```

Rows there mean the script has been failing and continuing. Your conversion will
turn those into visible failures — which is correct, and will look like the
conversion broke something. Predict it: [H11](../H-verification/H11-differences-that-should-exist.md).

---

Previous: [C7 · `WHILE` / `LOOP` iteration](C7-loops.md) ·
Next: [C9 · `BEGIN TRANSACTION` / `COMMIT`](C9-transactions.md)
