# G11 · Retry and partial-failure semantics

> **Part G — Scheduling, parameters, backfills** · Sourcing: `CRAFT`
> **The question:** the script stopped at the first error. What does dbt do?

Keeps going. It skips the failed model's **descendants** and builds everything
else. That's usually better, and it's a change worth understanding before it
happens unattended.

## The difference

```bash
set -e
bq query < load_users.sql      # fails here
bq query < load_orders.sql     # never runs
bq query < build_summary.sql   # never runs
```

Everything after the failure is skipped, related or not. You get a clean stop and
a half-updated warehouse.

```bash
dbt build --select +publish
```

`load_users` fails ⇒ its descendants are **skipped**; `load_orders` and its
branch **still build**. You get a partially-updated warehouse where the parts that
could succeed did.

## The statuses

dbt-core's `RunStatus` distinguishes more than pass/fail:

```
success · error · skipped · partial success · no-op · reused
```

`skipped` is the important one for this page — it means "an ancestor failed", not
"something went wrong here". A run log full of `skipped` nodes points at one real
failure upstream.

`partial success` appears for microbatch models where some batches succeeded and
others didn't ([G6](G6-backfill-microbatch.md)).

## Is partial success acceptable?

This is the question to answer during conversion, not during an incident.

**Usually yes.** Independent branches genuinely are independent, and building
what you can is better than building nothing.

**Sometimes no.** If downstream consumers assume the whole set is consistent — a
report joining three marts, an export expecting all of them fresh — a partial
build produces coherent-looking, wrong output.

If you need all-or-nothing:

```bash
dbt build --select +publish --fail-fast
```

`--fail-fast` stops at the first failure, closer to `set -e`. You still have
whatever was built before the stop; it just doesn't continue.

Neither option gives you transactional rollback across models. That doesn't exist
— [C9](../C-structural/C9-transactions.md).

## Retrying

```bash
dbt retry
```

Reruns from the last failure, using the previous `run_results.json` — successful
nodes aren't rebuilt. Much better than rerunning the whole script, which is what
the cron equivalent did.

Or explicitly:

```bash
dbt build --select result:error+ --state ./last-run
```

**Retry is only safe if your models are idempotent**, which is why
[E7](../E-translation/E7-idempotency-meaning.md) and
[E8](../E-translation/E8-idempotency-proving.md) matter. A model that appends
without a `unique_key` duplicates on every retry.

That's the connection people miss: automated retry makes non-idempotency an
active problem rather than a theoretical one.

## Where retries belong

**Transient failures** — quota, network, a flaky upstream — belong to your
orchestrator, which reruns the invocation.

**Deterministic failures** — bad SQL, a failing test, missing permissions — won't
be fixed by retrying, and retrying wastes money. Configure retry counts low
enough that a real failure surfaces quickly.

dbt itself retries some adapter-level operations, but it does not retry a model
that failed.

## Record the change

Add it to the baseline, because the failure behaviour genuinely differs:

```
Failure semantics:
  Old: set -e, stopped at first error, later steps never ran.
  New: dbt build continues; failed node's descendants skipped, other
       branches build. Partial success is possible.
  Decision: acceptable — the three marts are read independently.
       Reviewed with reporting team 2026-09-02.
```

Without that, the first partial failure looks like a conversion bug rather than a
documented behaviour change.

---

Previous: [G10 · State-based selection](G10-state-selection.md) ·
Next: [I1 · Conversion order](../I-migration/I1-conversion-order.md)
