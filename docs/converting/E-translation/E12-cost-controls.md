# E12 · Cost controls after conversion

> **Part E — Statement-level translation** · Sourcing: `CRAFT`
> **The question:** my script capped its own spend. Does that survive?

Only if you carry it. And a conversion changes the cost profile enough that the
old cap may be the wrong number.

## The pattern

```sql
SET @@maximum_bytes_billed = 100000000000;   -- 100 GB
```

A guard rail: if the query would read more than that, it fails instead of running.
Turns a runaway into an alert rather than a bill.

## Where it goes

**Profile-wide**, in `profiles.yml`, which is usually right:

```yaml
my_project:
  outputs:
    prod:
      type: bigquery
      maximum_bytes_billed: 1000000000000   # 1 TB
```

**Per model**, for one known-expensive model:

```sql
{{ config(pre_hook="set @@maximum_bytes_billed = 100000000000") }}
```

Legitimate pre-hook use — [F8](../F-hooks/F8-pre-hook-patterns.md).

Note the scope caveat from [E9](E9-session-settings.md): a pre-hook setting
applies to that model's connection, not the whole run.

## Why the old number may be wrong

The cap was sized for the script's behaviour. Conversion changes it, in both
directions:

**Lower, usually.** That's the point — an incremental model reads less than a full
rebuild. A cap sized for the old full scan won't catch a runaway incremental
model, because the runaway is still smaller than the original.

**Higher, sometimes.** The first run of an incremental model is a full build. A
cap sized for steady-state incremental reads will fail that first run, and every
`--full-refresh` after it.

That second case catches people during cutover: the model works in testing on a
small slice, then the production first-build hits the cap and fails.

## Set it from measurement

Take the baseline figures from [A9](../A-assess/A9-correctness-baseline.md) and
compare after conversion:

```sql
select
    date(creation_time) as d,
    destination_table.table_id,
    max(total_bytes_processed) / pow(10, 9) as max_gb
from `region-eu`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
where creation_time > timestamp_sub(current_timestamp(), interval 30 day)
  and destination_table.dataset_id = 'analytics'
group by 1, 2
order by 1 desc;
```

Set the cap at a comfortable multiple of the observed steady-state maximum — high
enough not to fire on a normal busy day, low enough to catch a model that has
stopped pruning.

## The failure this catches

The one worth having a cap for: a predicate that stops pruning.

Wrap a partition column in a function and the query still returns the right
answer, reading the whole table. No error, no warning — just cost. See
[the beginner track](../../beginner/04-partitioning-explained.md#what-your-query-prunes-actually-requires).

A cap turns that from a silent bill into a failed run, which is the outcome you
want. It's the single best argument for setting one.

## Don't cap so low it becomes noise

A cap that fires on ordinary variation gets raised until it's meaningless, or
worse, gets removed. Size it to catch a step change, not a busy Tuesday.

And exclude the first build from the reckoning — either run the initial build
with the cap raised deliberately, or set it after the model is established.

## The other controls

`maximum_bytes_billed` is the direct one. Also worth knowing:

- **Project-level quotas** — a hard ceiling per day, independent of dbt
- **Reservations / slots** — flat-rate rather than on-demand pricing; caps
  concurrency instead of bytes
- **`partition_expiration_days`** — storage cost, not query cost —
  [D7](../D-data-movement/D7-table-options.md)

Query cost is what conversion changes most, so that's where the guard belongs.
Track it afterwards — [J1](../J-operating/J1-cost-after-conversion.md).

---

Previous: [E11 · Query parameters → vars](E11-query-parameters.md) ·
Next: [E13 · Time travel](E13-time-travel.md)
