# C12 · Stored procedures calling other procedures

> **Part C — Structural archetypes** · Sourcing: `CRAFT`
> **The question:** we have a procedure that calls four others. Where do I even start?

At the leaves. A call tree is a DAG someone wrote as procedure calls, and the
conversion is mostly recovering that graph.

## The pattern

```sql
CREATE PROCEDURE analytics.run_daily()
BEGIN
    CALL analytics.load_users();
    CALL analytics.load_orders();
    CALL analytics.build_summary();
    CALL analytics.publish();
END;
```

The parent procedure isn't doing work — it's expressing order. Which is what a
DAG does, better.

## Map the tree first

Before converting anything, get the full call graph. Procedures nest, and the one
you were shown is rarely the whole story:

```sql
select routine_name, routine_definition
from `project.analytics.INFORMATION_SCHEMA.ROUTINES`
where routine_type = 'PROCEDURE';
```

Then grep the definitions for `CALL`:

```bash
grep -ioE 'call\s+[a-z0-9_.`]+' procedures.sql | sort -u
```

You want a tree, with each leaf's inputs and outputs recorded — that's
[A2](../A-assess/A2-map-dependencies.md) applied to procedures.

## Convert leaves first

The leaf procedures do the actual work. Each becomes a model (or several —
[C4](C4-fan-out.md)), classified by
[A3](../A-assess/A3-classify-by-write-pattern.md) like any other script.

Then the parent disappears:

```sql
-- was: CALL run_daily()
dbt build --select +publish
```

`+publish` means "publish and everything it depends on". The ordering that lived
in the parent procedure's body is now derived from `ref()` —
[E2](../E-translation/E2-ordering-by-ref.md).

## What the parent was hiding

Converting the tree usually surfaces things:

**Order that wasn't a dependency.** `load_users` before `load_orders` may be
arbitrary. Once converted they run in parallel, which is faster and reveals
whether the order mattered.

**Order that was a dependency but isn't declared.** `build_summary` reads what
`load_orders` wrote, but only line order enforced it. Now it's a `ref()`, and if
you miss it you get intermittent failures —
[E5](../E-translation/E5-finding-hardcoded-names.md).

**Procedures called from more than one parent.** Shared logic, currently
duplicated in the schedule. Becomes one model with two downstream consumers.

**Dead procedures.** Defined, never called. Check before converting:

```sql
select user_email, max(creation_time) as last_call
from `region-eu`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
where creation_time > timestamp_sub(current_timestamp(), interval 90 day)
  and query like '%load_users%'
group by 1;
```

No rows in 90 days ⇒ don't convert it — [A7](../A-assess/A7-what-not-to-convert.md).

## Parameters make it harder

```sql
CALL analytics.load_region('eu');
CALL analytics.load_region('us');
```

One procedure, two invocations. Options:

- **One model with a `union all`** over the regions, generated in Jinja —
  [C10](C10-dynamic-sql.md)
- **Separate models per region**, if they genuinely differ
- **One model with region as a column**, which is usually right and often what
  the data wanted anyway

The third is the answer most of the time. Per-region tables are frequently a
legacy of per-region loading, not a modelling decision.

## Convert incrementally

Don't attempt the whole tree in one change. Convert one leaf, declare its output
as a `source` for everything still calling the old procedures
([E3](../E-translation/E3-ref-vs-source.md)), and let the parent keep running.

Then the next leaf. The parent procedure shrinks until it's empty, and you delete
it. That's the strangler pattern —
[I2](../I-migration/I2-strangler-pattern.md) — and it's the only sane way to
handle a deep call tree.

---

Previous: [C11 · `CREATE TEMP FUNCTION`](C11-temp-functions.md) ·
Next: [C13 · Python scripts](C13-python-scripts.md)
