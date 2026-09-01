# K3 · Incremental applied where a table belongs

> **Part K — Anti-patterns** · Sourcing: `CRAFT`
> **The question:** the table is big, so shouldn't it be incremental?

Not necessarily. Incremental buys cost and sells correctness guarantees, and
people make that trade without pricing either side.

## What you give up

A `table` model recomputes from source every run. It therefore **cannot drift** —
whatever it produces is what your SQL says, every time.

Making it incremental introduces every failure mode in this documentation:

- Partitions that should be empty but aren't —
  [B14](../B-write-patterns/B14-when-the-range-can-empty.md)
- Duplicates from key handling —
  [B8](../B-write-patterns/B8-merge-on-clause-to-unique-key.md)
- Columns silently dropped — [D8](../D-data-movement/D8-add-column-migrations.md)
- Rows lost outside the lookback window —
  [G8](../G-scheduling/G8-late-arriving-data.md)

All silent. All requiring [monitoring](../J-operating/J2-monitoring-drift.md) you
now have to build and own.

## The arithmetic people skip

Work out what the saving actually is:

```sql
select
    round(avg(total_bytes_processed) / pow(10,12), 3) as avg_tb,
    round(avg(total_bytes_processed) / pow(10,12) * 5, 2) as avg_usd_at_5_per_tb
from `region-eu`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
where creation_time > timestamp_sub(current_timestamp(), interval 30 day)
  and destination_table.table_id = 'my_table';
```

A model rebuilding 200 GB nightly costs about £1 a day. Making it incremental
might save 80% of that — call it £290 a year — against an ongoing obligation to
monitor for silent corruption.

For a genuinely large model the saving is thousands and the trade is obvious. For
a small one it's a bad deal, and "it's big" is not the same as "it's expensive".

## The threshold

Rough guidance, not a rule:

| Full rebuild | Materialization |
| --- | --- |
| Seconds, pennies | `table`. Don't think about it |
| Under a minute, under £1/run | `table` unless run very frequently |
| Minutes, meaningful cost | Consider incremental |
| Doesn't finish, or genuinely expensive | Incremental, definitely |

Also weigh **how often it runs**. A £1 rebuild hourly is £8,760 a year; nightly
it's £365.

## Signals it should be a table

- The source is a full snapshot rather than an event stream
- Rows change unpredictably far back in history
- There's no natural partition column
- There's no reliable `unique_key`
- Correctness matters more than cost, and you can afford the rebuild
- The team is small and won't maintain drift monitoring

That last one is real. An incremental model is an ongoing commitment, not a
one-time optimisation.

## Convert to `table` first, incremental later

Two changes with different risk profiles. `table` first proves the SQL is right;
incremental afterwards is then a pure optimisation you can verify against the
table version.

Doing both at once is [K11](K11-convert-and-optimise.md), and it means a
discrepancy could be either the logic or the strategy.

## Going back is allowed

If an incremental model keeps drifting and the rebuild is affordable, changing it
back is a legitimate outcome, not a defeat. `materialized='table'`, delete the
`is_incremental()` block, and reclaim the guarantee.

Worth revisiting deliberately — [J9](../J-operating/J9-revisiting-strategy.md).

---

Previous: [K2 · Hooks as an escape hatch](K2-hooks-as-escape-hatch.md) ·
Next: [K4 · `run-operation` as a scheduler](K4-run-operation-as-scheduler.md)
