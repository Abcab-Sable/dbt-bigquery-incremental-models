# C3 · Multi-statement script → separate models

> **Part C — Structural archetypes** · Sourcing: `CRAFT`
> **The question:** when should a script's steps become several models rather than one?

When the intermediate results are worth naming. That's the test, and it's more
often true than people expect during a conversion.

## The signals

Split a step into its own model when **any** of these hold:

**Something else needs it.** Even one other consumer justifies a model — it's the
difference between a shared definition and two copies that drift.

**It's expensive and reused.** Computed once, read many times. A CTE or an
ephemeral model recomputes per reference; a table doesn't.

**It's a meaningful business concept.** "Active users", "valid orders",
"deduplicated events" are things people talk about. Things people talk about
should have names.

**You want to test it.** A CTE can't be tested. If the correctness of the
intermediate matters — and after a conversion it usually does — it needs to be a
model.

**It would make the parent model too big.** Ten CTEs is a smell.

**Different refresh cadences.** If one step needs hourly and another daily,
they cannot be one model.

## The conversion

```sql
CREATE TEMP TABLE cleaned AS SELECT ... FROM raw.events WHERE ...;
CREATE TEMP TABLE enriched AS SELECT ... FROM cleaned JOIN dim_users ...;
CREATE OR REPLACE TABLE analytics.daily_summary AS SELECT ... FROM enriched GROUP BY ...;
```

Becomes three files:

```
models/staging/stg_events.sql        ← cleaned
models/intermediate/int_events_enriched.sql   ← enriched
models/marts/daily_summary.sql       ← the output
```

Each `ref()`s the previous. Ordering is derived, not stated —
[E2](../E-translation/E2-ordering-by-ref.md).

## Choosing materializations per layer

You now get to make a decision the script never could:

| Layer | Usually |
| --- | --- |
| Staging (`stg_`) | `view` or `ephemeral` — cheap, thin |
| Intermediate (`int_`) | `view`, or `table` if expensive and reused |
| Marts | `table` or `incremental` |

Only the mart usually needs incremental. Making every layer incremental
multiplies the failure modes in this documentation by the number of layers, for
no benefit.

## The cost you're accepting

More models means more tables (or views), more objects in the dataset, more
things to name. In exchange you get lineage, testability, independent
scheduling, and the ability to rebuild one step without the others.

For a conversion, the tie-breaker is usually **debuggability**. When parity fails
in [H4](../H-verification/H4-column-level-diffing.md), separate models let you
compare each stage against the script's corresponding temp table. One giant model
gives you one place to look and no way to narrow it.

That alone often justifies splitting during conversion, even if you'd collapse
some of it later.

## Don't split mechanically

The opposite failure is one model per script statement, including the trivial
ones. A `SELECT` that filters two rows doesn't need a name, a file, and a table.
That's [K8](../K-antipatterns/K8-one-model-per-statement.md).

Ask what each step *is*, not just that it exists.

## Naming

Whatever convention you use, be consistent, and prefix by layer (`stg_`, `int_`,
`fct_`, `dim_`). Conversions produce a lot of intermediate models at once, and
without a convention the project becomes unreadable quickly.

Keep the script's own names where they were meaningful. `cleaned` →
`stg_events_cleaned` preserves the connection for whoever diffs the two.

---

Previous: [C2 · Ephemeral models](C2-ephemeral-models.md) ·
Next: [C4 · Scripts writing to several tables](C4-fan-out.md)
