# J1 · Cost after conversion: what changed, and how to see it

> **Part J — Operating it afterwards** · Sourcing: `CRAFT`
> **The question:** did the conversion actually save money?

Measure it. The saving is usually real and usually smaller than expected, and the
places it fails to appear are diagnostic.

## The comparison

Baseline figures come from [A9](../A-assess/A9-correctness-baseline.md). Compare
against the same query afterwards:

```sql
select
    date(creation_time) as d,
    destination_table.table_id,
    count(*) as jobs,
    round(sum(total_bytes_processed) / pow(10,12), 3) as tb_processed,
    round(avg(timestamp_diff(end_time, start_time, second)), 1) as avg_seconds
from `region-eu`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
where creation_time > timestamp_sub(current_timestamp(), interval 30 day)
  and destination_table.dataset_id = 'analytics'
group by 1, 2
order by 1 desc, tb_processed desc;
```

Compare like with like: a steady-state incremental day against a steady-state
script day. Don't include the first build.

## Where the saving comes from

Not from "dbt is faster". From:

**Reading less.** An incremental model reads new data; the script read everything.
This is the big one, and it's the whole argument.

**Fewer full rebuilds.** A script rebuilding nightly does 365 full scans a year.

**Parallelism reducing wall-clock**, though not bytes — relevant for slot-based
pricing, not on-demand.

## Where it fails to appear

Five diagnoses, in the order worth checking:

**`merge` on an unpartitioned table.** The merge scans the target to find matches,
so cost scales with table size regardless of how little new data there is. Add
partitioning and `incremental_predicates` —
[B12](../B-write-patterns/B12-extra-predicates.md).

**A predicate that doesn't prune.** A partition column wrapped in a function
reads everything, silently. The commonest cause of "why is this still expensive"
— [the beginner track](../../beginner/04-partitioning-explained.md#what-your-query-prunes-actually-requires).

**`on_schema_change` forcing a temp table.** Anything other than `ignore`
materialises your model output before the merge, turning one statement into two.
Usually worth it; measure it —
[D8](../D-data-movement/D8-add-column-migrations.md).

**A lookback window wider than needed.** Seven days reprocessed where two would
do costs 3.5× more, forever — [G8](../G-scheduling/G8-late-arriving-data.md).

**`is_incremental()` not actually applying.** Read `target/compiled/` and check
the filter is present. A misplaced `{% if %}` produces a full scan every run and
looks fine.

## New costs the script didn't have

Be honest in the comparison:

- **CI builds.** Every pull request may build models —
  [G10](../G-scheduling/G10-state-selection.md) is how you keep this small.
- **Dev environments.** Each developer building their own copies.
- **Tests.** They're queries, and they cost.
- **Shadow running.** Double cost for the duration — temporary but real.

A conversion that halves production cost while tripling CE cost has not
necessarily won.

## Attribute it

Label your tables so cost can be split by team or model —
[D7](../D-data-movement/D7-table-options.md). Without labels, all you have is one
dataset-level number and no way to find the expensive model.

## Check it more than once

Cost drifts. Data volumes grow, someone widens a window, a predicate stops
pruning after a refactor. Make this query a scheduled report rather than a
one-off, and look at the trend — [J9](J9-revisiting-strategy.md).

---

Next: [J2 · Monitoring incremental drift](J2-monitoring-drift.md) ·
Back to [the backlog](../BACKLOG.md)
