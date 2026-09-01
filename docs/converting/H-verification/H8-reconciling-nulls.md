# H8 · Reconciling: null vs empty string vs absent

> **Part H — Proving correctness** · Sourcing: `CRAFT`
> **The question:** old has `''`, new has `NULL`. Which is right, and does it matter?

Usually it matters, because downstream filters treat them differently. And the
comparison itself needs care, since `NULL != NULL` is `NULL`, not `true`.

## Compare with `IS DISTINCT FROM`

The trap first:

```sql
-- WRONG: returns NULL when either side is null, so the row isn't counted
where old.status != new.status
```

A row where old is `NULL` and new is `'active'` does **not** satisfy `!=`. It
evaluates to `NULL`, the `WHERE` drops it, and your diff silently undercounts.

```sql
-- right
where old.status is distinct from new.status
```

`IS DISTINCT FROM` treats two nulls as equal and null-vs-value as different.
That's what you want for every comparison in [H4](H4-column-level-diffing.md).

## Find the pattern

```sql
select
    countif(old.status is null and new.status = '')     as old_null_new_empty,
    countif(old.status = ''    and new.status is null)  as old_empty_new_null,
    countif(old.status is null and new.status is not null and new.status != '') as old_null_new_value,
    countif(old.status is not null and new.status is null) as old_value_new_null
from analytics.orders old
full outer join analytics_dbt.orders new using (order_id);
```

The shape tells you the cause:

| Pattern | Cause |
| --- | --- |
| Consistently null ↔ empty string | A `COALESCE` or `NULLIF` that moved, or a cast difference |
| Old null, new has values | You removed a filter, or fixed a join |
| Old has values, new null | A join now dropping rows — usually `LEFT` became `INNER` |
| Scattered, no pattern | Non-determinism, or an ordering-dependent tiebreak |

## Where the difference comes from

**A lost `COALESCE`.** The script had `COALESCE(status, '')` and the model
doesn't, or vice versa. Check the original SQL character by character — this is
the single most common cause.

**Join type changed.** Rewriting a script's implicit join into explicit `JOIN`
syntax is where `LEFT` quietly becomes `INNER`. Rows vanish rather than becoming
null, so it shows as missing rows *and* null differences.

**Aggregate over an empty set.** `SUM()` of nothing is `NULL`; `COUNT()` of
nothing is `0`. If the grouping changed, so does which you get.

**`SAFE_CAST` vs `CAST`.** `SAFE_CAST` yields `NULL` on failure where `CAST`
errors. Introducing one to "make it robust" converts errors into nulls, which is
a behaviour change.

## Which is correct?

Ask what downstream does with it:

```sql
where status != 'cancelled'
```

This **excludes nulls**. If the old table had `''` and the new has `NULL`, rows
that used to pass this filter now don't — the conversion changed downstream
results without changing the column's meaning.

That's the real risk. The difference between `NULL` and `''` is invisible in a row
count and decisive in a `WHERE` clause.

## Decide and record

Either bar is fine as long as it's explicit:

**Match the old exactly** — restore the `COALESCE`, accept the semantics you
inherited.

**Fix it** — nulls where nulls are meant. But that's a deliberate behaviour
change: predict it, record it in [H11](H11-differences-that-should-exist.md), and
tell downstream consumers ([I5](../BACKLOG.md#part-i--migration-strategy)).

What you must not do is notice the difference, decide it's cosmetic, and move on.

## Lock it in

```yaml
models:
  - name: orders
    columns:
      - name: status
        data_tests:
          - not_null
```

If the column should never be null, assert it. Then the question can't recur
silently — [H12](H12-tests-from-guarantees.md).

---

Previous: [H7 · Reconciling: ordering](H7-reconciling-ordering.md) ·
Next: [H9 · Reconciling: numeric precision](H9-reconciling-numeric-precision.md)
