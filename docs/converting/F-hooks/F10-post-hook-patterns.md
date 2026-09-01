# F10 · post-hook patterns worth keeping

> **Part F — Hooks** · Sourcing: `CRAFT`
> **The question:** what belongs in a post-hook?

Things that are *about* the relation you just built, but not *part of* it. That's
a narrower set than it sounds, and the useful ones are mostly metadata.

## The legitimate ones

**Table options** — description, labels, expiration:

```sql
{{ config(post_hook="""
    alter table {{ this }} set options(
        description = 'Daily event counts by user',
        labels = [('team', 'analytics'), ('tier', 'gold')]
    )
""") }}
```

Detail in [F12](F12-post-hook-table-options.md). Note that description is often
better handled by `persist_docs` — which runs *after* your hook
([F4](F4-where-hooks-run.md)), so if you set both, dbt wins.

**Audit rows** recording that this model ran:

```sql
{{ config(post_hook="{{ log_model_run(this) }}") }}
```

Legitimate, with caveats about scale and alternatives —
[F13](F13-post-hook-audit-rows.md).

**Environment-gated operations** that render empty elsewhere via
[F3](F3-empty-hook-skipping.md).

**Cleaning up something the model deliberately created.** Rare, and if it comes
up often your model is doing too much.

## What to check before writing one

At post-hook time:

- the model's output **exists** and is complete
- `apply_grants` has **not** run — [F11](F11-grants-vs-post-hook.md)
- `persist_docs` has **not** run — so it will overwrite a description you set
- the **temp relation may still exist** — [F15](F15-hooks-and-temp-relation.md)
- on microbatch, this runs on the **last batch only** — [F16](F16-hooks-and-failure.md)

## Keep them idempotent

A post-hook runs every time the model does, including retries and backfills. If
running it twice is different from once, you have the same problem the model
would have — [E7](../E-translation/E7-idempotency-meaning.md).

`ALTER TABLE ... SET OPTIONS` is idempotent. `INSERT INTO audit_log` is not, and
that's usually tolerable for an audit trail — but decide, rather than discovering
it after a backfill writes 400 rows.

## Keep them cheap

The hook runs inside the model's build. A slow post-hook makes the model slow,
and it shows up in the run as time attributed to the model rather than to the
hook. A post-hook that scans a large table is a hidden cost on every build.

If a post-hook needs to query data to decide what to do, that's a signal the work
belongs in a model.

## Keep them one statement

Prefer a macro over a list ([F7](F7-hook-ordering.md)). A list has partial-failure
states; a single statement doesn't. And a macro is testable, which a config string
is not.

## The ones to reject

| Tempting post-hook | Better |
| --- | --- |
| `INSERT`/`MERGE` into another table | A model — [F17](F17-when-a-hook-is-wrong.md) |
| `GRANT` | The `grants` config — [F11](F11-grants-vs-post-hook.md) |
| `EXPORT DATA` to GCS | Orchestration — [D2](../D-data-movement/D2-export-data.md) |
| Send a notification | Your scheduler |
| Delete old rows for retention | A separate scheduled operation |
| Refresh a downstream table | `ref()` and let the DAG do it |
| Run data-quality checks | dbt tests — [H12](../H-verification/H12-tests-from-guarantees.md) |

The pattern in that right-hand column: almost every rejected post-hook has a
first-class dbt feature that does the job with lineage, retries, and visibility.

## The test

> **If this fails, should the model be considered failed?**

Yes ⇒ it's arguably part of the model, and probably belongs in the model.
No ⇒ it shouldn't be attached to the model's build at all.

Only things that sit awkwardly between those — metadata, audit — are genuinely
post-hook shaped.

---

Previous: [F9 · pre-hook deletes](F9-pre-hook-deletes.md) ·
Next: [F11 · post-hook vs the `grants` config](F11-grants-vs-post-hook.md)
