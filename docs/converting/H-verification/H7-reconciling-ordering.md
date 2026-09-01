# H7 · Reconciling: row and column ordering

> **Part H — Proving correctness** · Sourcing: `CRAFT`
> **The question:** the rows are the same but in a different order. Does it matter?

Almost never. Fix the comparison, not the model.

## Row order is not a property of a table

BigQuery tables have no inherent order. Two queries returning the same rows in
different sequences returned the same result. If your comparison says otherwise,
your comparison is wrong.

Use order-independent techniques:

```sql
-- order-independent: set difference
select * from old except distinct select * from new;

-- order-independent: sum of fingerprints
select sum(farm_fingerprint(to_json_string(t))) from old t;
```

Not:

```sql
-- meaningless
select * from old limit 10;   -- compared by eye against new
```

Eyeballing the first ten rows of each is the commonest way people convince
themselves something is broken when it isn't.

## Column order can matter for the comparison

`EXCEPT DISTINCT` is positional — it compares column 1 to column 1. If your model
emits columns in a different sequence from the old table, the comparison fails
even when every value is right.

Two fixes. Project explicitly:

```sql
select event_date, user_id, event_count from analytics.daily_events
except distinct
select event_date, user_id, event_count from analytics_dbt.daily_events
```

Or match the original order in the model, which is worth doing anyway — a
diff-friendly model is one whose `select` mirrors the table people already know.

Check the original order from your baseline:

```sql
select column_name, ordinal_position
from `project.analytics.INFORMATION_SCHEMA.COLUMNS`
where table_name = 'daily_events'
order by ordinal_position;
```

## When order genuinely matters

Rare, but real:

**Clustering.** Physical ordering affects query cost, not correctness. If the old
table was clustered and your model isn't, results match while performance
doesn't. That's a config difference — [D6](../D-data-movement/D6-partitioning-ddl.md) —
and it won't show up in any parity check. Check it separately.

**A downstream consumer relying on order without `ORDER BY`.** They have a latent
bug. Your conversion may expose it. Worth a warning during cutover
([I5](../I-migration/I5-notifying-consumers.md)) even though it isn't your bug.

**Non-deterministic tiebreaks.** If a `QUALIFY ROW_NUMBER()` orders by a
non-unique column, *which* row survives varies between runs. That isn't an
ordering issue — it's non-determinism, and it makes the output genuinely
different each time. Fix it with a stable tiebreaker —
[B16](../B-write-patterns/B16-deduplication.md).

That last case is the one to watch. It presents as "just an ordering difference"
and is actually a correctness problem.

## Rule of thumb

If reordering the comparison makes the difference vanish, it was never a
difference. If it doesn't, you have a real one — go to
[H4](H4-column-level-diffing.md).

---

Previous: [H6 · How long to shadow](H6-shadow-duration.md) ·
Next: [H8 · Reconciling: nulls](H8-reconciling-nulls.md)
