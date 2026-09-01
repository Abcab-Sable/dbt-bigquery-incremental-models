# E8 · Idempotency: proving it

> **Part E — Statement-level translation** · Sourcing: `CRAFT`
> **The question:** how do I actually check this, rather than reasoning about it?

Run it twice and compare. The whole test takes two minutes, and it catches the
most common conversion bug there is.

## The two-run check

```bash
dbt run --select daily_events
```

```sql
select count(*) as after_first from analytics_dbt.daily_events;
```

```bash
dbt run --select daily_events
```

```sql
select count(*) as after_second from analytics_dbt.daily_events;
```

**Equal ⇒ pass. Higher ⇒ your model appends where you meant it to replace.**

Nearly always a missing `unique_key`, or a watermark window overlapping without
one. See [E7](E7-idempotency-meaning.md).

## Counts aren't quite enough

A count catches duplication. It won't catch rows *changing* between runs, which
is what non-deterministic generation does. Compare content:

```sql
-- before the second run
create table scratch.snap_1 as select * from analytics_dbt.daily_events;
```

```sql
-- after the second run
select
    (select count(*) from scratch.snap_1) as snap_rows,
    (select count(*) from analytics_dbt.daily_events) as live_rows,
    (select count(*) from (
        select * from scratch.snap_1
        except distinct
        select * from analytics_dbt.daily_events
    )) as in_snap_not_live,
    (select count(*) from (
        select * from analytics_dbt.daily_events
        except distinct
        select * from scratch.snap_1
    )) as in_live_not_snap;
```

All four numbers consistent — the two counts equal, both difference counts zero —
means genuinely identical output.

Non-zero differences with equal counts means rows *changed*: `generate_uuid()`,
`current_timestamp()`, or something else non-deterministic in the model body.

## Check for duplicates directly

Independent of run count, and worth doing once:

```sql
select order_id, count(*) as n
from analytics_dbt.orders
group by order_id
having count(*) > 1
order by n desc
limit 20;
```

For a composite key, group by all of it. This is the check that finds the
[nullable composite key](../B-write-patterns/B8-merge-on-clause-to-unique-key.md#single-key-vs-composite-not-the-same-code-path)
problem, because those rows re-insert on every run and the duplicate count climbs
run over run.

Make it permanent as a test:

```yaml
models:
  - name: orders
    columns:
      - name: order_id
        data_tests:
          - unique
          - not_null
```

## Test the empty case too

Idempotency and the empty-partition behaviour are different properties, and
passing one says nothing about the other. Run both checks:

- Two-run count check ⇒ catches duplication
- Forced-empty partition check ⇒ catches [B14](../B-write-patterns/B14-when-the-range-can-empty.md)

The procedure for the second is in
[H2](../H-verification/H2-row-count-parity.md#test-the-empty-case-deliberately).

## Was the old script idempotent?

Worth knowing before you compare. If the script wasn't, and someone occasionally
re-ran it, the baseline table may contain duplicates that your correct model
won't reproduce.

That's a legitimate difference, not a conversion bug — but only if you knew about
it in advance. It goes in [A9](../A-assess/A9-correctness-baseline.md), and it's
exactly the case [H11](../H-verification/H11-differences-that-should-exist.md) exists for.

## Put it in CI

```bash
dbt build --select daily_events
dbt build --select daily_events
dbt test --select daily_events
```

Building twice in CI makes non-idempotency a failed pipeline rather than a
discovery. Cheap, and it holds the property permanently instead of at one moment.

---

Previous: [E7 · Idempotency: what it means](E7-idempotency-meaning.md) ·
Next: [B1 · `CREATE OR REPLACE TABLE` → `table`](../B-write-patterns/B1-create-or-replace-to-table.md)
