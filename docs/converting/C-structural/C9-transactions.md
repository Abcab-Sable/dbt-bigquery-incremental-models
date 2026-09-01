# C9 · `BEGIN TRANSACTION` / `COMMIT`

> **Part C — Structural archetypes** · Sourcing: `SRC`
> **The question:** my script wraps everything in a transaction. What does dbt do?

Nothing equivalent, on BigQuery. dbt doesn't wrap models in transactions here, and
the atomicity you need comes from using a single statement rather than from a
transaction around several.

## What the script was buying

```sql
BEGIN TRANSACTION;

DELETE FROM analytics.daily_events WHERE event_date >= @start;
INSERT INTO analytics.daily_events SELECT ... ;

COMMIT TRANSACTION;
```

All-or-nothing across two statements. A failure between them rolls back, so the
table is never left with the delete applied and the insert missing.

## dbt's answer is one statement, not a transaction

The conversion makes the transaction unnecessary rather than reproducing it:

```sql
{{ config(
    materialized='incremental',
    incremental_strategy='insert_overwrite',
    partition_by={'field': 'event_date', 'data_type': 'date'},
    partitions=['date_sub(current_date(), interval 1 day)', 'current_date()']
) }}
```

The generated `MERGE` does the delete and the insert in **one statement**:

```sql
merge into <target> ... on FALSE
when not matched by source and <partition predicate> then delete
when not matched then insert (...) values (...)
```

A single `MERGE` is atomic in BigQuery. Same guarantee, no transaction —
[B13](../B-write-patterns/B13-delete-insert-to-insert-overwrite.md).

## `transaction: true` on hooks doesn't mean this

Hooks carry a `transaction` flag, and the name misleads:

```jinja
{% for hook in hooks | selectattr('transaction', 'equalto', inside_transaction) %}
```

It selects **which phase** a hook belongs to. On BigQuery every materialization
calls `run_hooks` only at the `True` default, so the flag doesn't wrap anything
in a transaction — there is no transaction. See
[F6](../F-hooks/F6-transaction-filter.md).

The default materialization for other warehouses emits `commit;` between phases;
BigQuery's doesn't.

## Where atomicity is genuinely lost

Some things the script had do not survive:

**Multi-table atomicity.** A script writing three tables in one transaction had
all-or-nothing across all three. Three dbt models don't — each is atomic
individually, and a failure can leave one built and two not.

There is no dbt mechanism for this. If it matters, the options are: combine into
one model where possible, or handle partial success in your orchestrator. Record
the decision — [C4](C4-fan-out.md) and
[G11](../G-scheduling/G11-retry-and-failure.md).

**Hooks with the model.** A post-hook is a separate statement; a failure after the
model builds leaves the model built and the hook unapplied, with no rollback —
[F16](../F-hooks/F16-hooks-and-failure.md).

**Anything you put in a pre-hook that modifies data.** Reintroduces the exact gap
the transaction closed. Don't — [F9](../F-hooks/F9-pre-hook-deletes.md).

## The check

For each transaction in the script, ask: **what would be inconsistent if this
failed halfway?**

- One table, delete + insert ⇒ solved by `insert_overwrite`
- One table, several inserts ⇒ solved by one `select` with `union all`
- Several tables ⇒ **not solved.** Decide explicitly what partial success means

The first two are the majority, and the conversion genuinely improves them —
you go from "atomic because we asked for a transaction" to "atomic because it's
one statement", which can't be accidentally undone by someone adding a step.

---

Previous: [C8 · `EXCEPTION WHEN ERROR`](C8-exception-handling.md) ·
Next: [C10 · `EXECUTE IMMEDIATE` and dynamic SQL](C10-dynamic-sql.md)
