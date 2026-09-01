# K9 · Ephemeral overuse

> **Part K — Anti-patterns** · Sourcing: `CRAFT`
> **The question:** ephemeral models give me names without tables. Why not use them everywhere?

Because the SQL is inlined into every consumer and re-run each time. Ephemeral
saves storage, not compute — and stacking them produces compiled SQL nobody can
debug.

## The shape

Eight ephemeral models chained:

```
stg_events (ephemeral)
  → int_cleaned (ephemeral)
    → int_enriched (ephemeral)
      → int_ranked (ephemeral)
        → daily_summary (table)
```

Only one relation exists. It looks elegant. The compiled `daily_summary` is four
levels of nested CTEs with generated `__dbt__cte__` names, and if two models
consume `int_enriched`, its whole upstream chain runs twice.

## What it costs

**Compute multiplies.** An ephemeral model referenced by three models is compiled
into all three and executed three times. Expensive logic in an ephemeral model is
paid for per consumer.

**Debugging gets hard.** You cannot `select *` from an ephemeral model — nothing
exists. To inspect stage three you compile a consumer and extract the CTE from
generated SQL.

**Testing is limited.** Tests are inlined too; `--store-failures` has nothing to
point at.

**Compiled output becomes unreadable.** Nested `__dbt__cte__` blocks, four deep,
in a file you're trying to compare against the original script during
[verification](../H-verification/H4-column-level-diffing.md).

That last one matters specifically during a conversion, when reading compiled SQL
is the main debugging tool.

## When ephemeral is right

- **Cheap** logic — a filter, a rename, a cast
- Shared by **several** models, so it deserves a name
- You don't need to query it directly
- **One level**, not a chain

Classic fit: a thin staging model standardising column names, used by three marts.

## When it isn't

| Situation | Use |
| --- | --- |
| Expensive and reused | `view` or `table` — compute once |
| Needed for debugging | `view` — free to materialise, queryable |
| Deep chain | `view` at least, so each level is inspectable |
| Heavily tested | `view` or `table`, so `store_failures` works |
| Only one consumer | A CTE in that consumer — [C1](../C-structural/C1-multi-statement-to-ctes.md) |

Views are underrated here. They cost nothing to store, they're queryable, and they
keep compiled SQL readable. For most intermediates during a conversion, `view` is
the better default and you can switch to `ephemeral` later if the object count
bothers you.

## The rule of thumb

**Ephemeral for cheap shared logic. Table for expensive shared results. View when
you're unsure.**

And: **one level of ephemeral, not a chain.** If a script's five-deep temp table
chain became five ephemeral models, that chain wants real intermediate models —
[C3](../C-structural/C3-separate-models.md).

## Spotting it in compiled output

```bash
dbt compile --select daily_summary
grep -c "__dbt__cte__" target/compiled/*/models/**/daily_summary.sql
```

More than two or three, and the model is carrying an ephemeral chain. Worth
checking during conversion, when you can still change it cheaply.

---

Previous: [K8 · One model per statement](K8-one-model-per-statement.md) ·
Next: [K10 · No tests, because the script had none](K10-no-tests.md)
