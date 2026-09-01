# 7. Where to go next

## What you now know

Reading the six pages before this, you've covered:

- Why incremental models exist: full rebuilds cost more every day, while the new
  data stays constant.
- The trade they make: cheaper, but capable of being **silently wrong** in ways a
  full rebuild can't be.
- `is_incremental()` and `{{ this }}` for filtering to new rows.
- Why `unique_key` decides whether you get updates or an append.
- Partitioning, pruning, and why `day` granularity is nearly always right.
- The three strategies, what each replaces, and when to reach for them.
- The failure modes, starting with the empty partition that stays full.

That's a working understanding. You can build these models and debug them.

## The habits worth keeping

**Read the compiled SQL.** In `target/compiled/`. It's the actual text sent to
BigQuery, and most incremental bugs are visible in it within thirty seconds. This
one habit replaces a lot of guessing.

**Compare against a full refresh.** Build the model both ways in a scratch
dataset and diff them. This is the only reliable way to catch silent drift, and
it's worth automating.

**Be suspicious of a run that always succeeds.** Most of the failures on
[page 6](06-when-things-go-wrong.md) never produce an error. Green runs are not
evidence of correct data.

**Write down what your `incremental_predicates` window assumes.** "Orders don't
change after 7 days" is a claim about your business. Put it in a comment. When it
stops being true, someone needs to be able to find it.

## Moving up a tier

When the explanations here start feeling slow, go to the **balanced track**. Same
material, written for someone who no longer needs partitioning explained:

- [1. How the materialization runs](../balanced/01-how-the-materialization-runs.md) —
  the exact branch order, and when dbt does or doesn't build a temp table
- [2. Choosing a strategy](../balanced/02-choosing-a-strategy.md)
- [3. The `merge` strategy](../balanced/03-merge.md) — the generated `MERGE` in
  full, and the composite-key null problem in detail
- [4. The `insert_overwrite` strategy](../balanced/04-insert-overwrite.md) — all
  four code paths, including `copy_partitions`
- [5. The `microbatch` strategy](../balanced/05-microbatch.md)
- [6. `partition_by` in detail](../balanced/06-partition-config.md) — how the
  config becomes SQL predicates
- [7. Schema changes](../balanced/07-schema-changes.md) — including BigQuery's
  `STRUCT` handling
- [8. Gotchas](../balanced/08-gotchas.md) — everything from page 6 here, stated
  compactly with source references

The [expert track](../expert/README.md) is the same knowledge compressed into a
reference you'd scan, not read. It assumes all of the above. It's where you go
when you know the material and need to check one specific thing.

## What this documentation deliberately doesn't cover

**The batching machinery for `microbatch`.** `event_time`, `begin`, `lookback`,
and how dbt splits a run into batches all live in dbt-core, a different codebase
that wasn't read for this. Everything else here was verified against source;
that part wasn't, and it's flagged wherever it comes up.

**Other data warehouses.** Snowflake, Redshift, Databricks and the rest implement
the same *strategy names* with genuinely different SQL. Don't carry these
conclusions across — the differences are not cosmetic.

**Other materializations.** Snapshots, seeds, and materialized views solve
adjacent problems and behave differently.

## Where the facts came from

Every claim in this documentation was checked against the dbt adapter source
code, pinned to a specific commit and package version recorded in
[the repository README](../../README.md). Where the released package and the
development branch differ, both are described.

That's the point of this repo. The official docs describe intent; these pages
describe what the code does. When the two disagree, the code wins — and now you
know where to look.

---

Previous: [6. When things go wrong](06-when-things-go-wrong.md) ·
Back to [the beginner index](README.md) ·
Up to [the repository README](../../README.md)
