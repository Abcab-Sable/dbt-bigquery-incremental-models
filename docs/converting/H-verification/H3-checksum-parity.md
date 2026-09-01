# H3 · Checksum and hash parity

> **Part H — Proving correctness** · Sourcing: `CRAFT`
> **The question:** same rows, not just the same number of them?

Hash each row, compare the sets. One query, and it upgrades you from
[definition 1 to definition 2](H1-what-correct-means.md).

## The set comparison

BigQuery's `EXCEPT DISTINCT` does the work directly — no hashing needed for a
first pass:

```sql
select
    (select count(*) from (
        select * from analytics.daily_events
        except distinct
        select * from analytics_dbt.daily_events
    )) as in_old_not_new,
    (select count(*) from (
        select * from analytics_dbt.daily_events
        except distinct
        select * from analytics.daily_events
    )) as in_new_not_old;
```

Both zero ⇒ the two tables contain identical row sets.

Run **both directions**. One alone tells you half the story, and the half you
skip is usually the interesting one.

## When column order or types differ

`EXCEPT DISTINCT` requires matching column order and compatible types. If they
differ, project explicitly:

```sql
with old as (
    select event_date, user_id, event_count from analytics.daily_events
),
new as (
    select event_date, user_id, event_count from analytics_dbt.daily_events
)
select
    (select count(*) from (select * from old except distinct select * from new)) as in_old_not_new,
    (select count(*) from (select * from new except distinct select * from old)) as in_new_not_old;
```

Listing columns explicitly is better practice anyway — it makes the comparison's
scope visible and stops a new column silently changing what you tested.

## Per-partition hashing

A single set comparison tells you *whether* they differ. Hashing per partition
tells you *where*, which is what you need to act:

```sql
with old as (
    select event_date,
           count(*) as n,
           sum(farm_fingerprint(to_json_string(t))) as fp
    from analytics.daily_events t
    group by event_date
),
new as (
    select event_date,
           count(*) as n,
           sum(farm_fingerprint(to_json_string(t))) as fp
    from analytics_dbt.daily_events t
    group by event_date
)
select
    coalesce(old.event_date, new.event_date) as event_date,
    old.n as old_n, new.n as new_n,
    old.fp = new.fp as fingerprints_match
from old full outer join new using (event_date)
where old.n is distinct from new.n
   or old.fp is distinct from new.fp
order by event_date;
```

`sum()` of fingerprints is order-independent, so row ordering can't produce a
false mismatch — [H7](H7-reconciling-ordering.md).

The result is a list of partitions to investigate, which is a far better starting
point than "the tables differ".

## The precision caveat

`to_json_string` serialises floats in a way that makes tiny representation
differences produce completely different hashes. A `FLOAT64` sum computed in a
different order can differ in the last bit and fail this check while being
correct.

If the tables contain floats and this check fails, don't debug the hash — go
straight to [H9](H9-reconciling-numeric-precision.md) and compare with a
tolerance. Hashing is the wrong tool for approximate types.

Same applies to timestamps with sub-second precision —
[H10](H10-reconciling-timestamps.md).

## Excluding non-deterministic columns

Anything generated at run time will never match:

```sql
select * except(_loaded_at, _run_id) from analytics.daily_events
```

Exclude them explicitly and **record that you did** in your
[H1](H1-what-correct-means.md) scope. An unstated exclusion is the difference
between a proof and a claim.

If a column you *want* to compare turns out to be non-deterministic, that's a
finding, not an inconvenience — see
[E7](../E-translation/E7-idempotency-meaning.md).

## What to do with a mismatch

You now know **which partitions** differ but not which **columns**. That's
[H4](H4-column-level-diffing.md), and it's the natural next step once this
narrows the search.

---

Previous: [H1 · What "correct" means](H1-what-correct-means.md) ·
Next: [H4 · Column-level diffing](H4-column-level-diffing.md)
