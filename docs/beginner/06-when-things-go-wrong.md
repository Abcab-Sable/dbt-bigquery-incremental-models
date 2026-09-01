# 6. When things go wrong

Each of these is a real failure mode. Symptom first, since that's what you'll
have when you arrive here.

## The partition that would not empty

**Symptom.** Your table shows rows that shouldn't exist. A day's data was
retracted upstream, your model correctly produces nothing for that day, and the
old rows are still sitting there. Every run "succeeds".

**Walk through it.** You have an events model using `insert_overwrite`,
partitioned by day.

Monday's run: the model produces 1,000 rows for `2026-08-31`. dbt looks at the
output, sees `2026-08-31`, replaces that partition. Correct.

Tuesday: upstream discovers Monday's data was garbage and deletes it. Your model
now correctly produces **zero rows** for `2026-08-31`.

dbt looks at your output to decide which partitions to replace. Your output is
empty. There are no partitions in it. So the replacement list is empty, the
delete clause matches nothing, and **the 1,000 stale rows stay exactly where they
are.**

The run succeeds. Row counts look plausible. Nothing is logged. You find out when
someone asks why the numbers don't match the source.

**Cause.** From [page 5](05-the-three-strategies.md#what-it-actually-runs), dbt
builds the list of partitions to replace by reading the rows your model produced:

```sql
set (dbt_partitions_for_replacement) = (
    select as struct array_agg(distinct event_date IGNORE NULLS)
    from tmp_events
);
```

No rows → no partitions → nothing deleted.

This is not a bug. `insert_overwrite` means "overwrite the partitions I produced".
It has no concept of the partitions you *meant to empty*. People read it as "make
the table match my model", which is what a `table` materialization does and this
does not.

**Fixes**, best first:

1. **List the partitions explicitly.** dbt then overwrites them regardless of
   whether your model produced rows:
   ```sql
   partitions=['date_sub(current_date(), interval 1 day)', 'current_date()']
   ```
2. **Make emptiness explicit.** Join to a date spine so every day you care about
   always has at least one row.
3. **Full refresh periodically** to reconcile drift, accepting the cost.

**Related.** Rows with a `NULL` partition value have the same problem for a
different reason. The `IGNORE NULLS` above excludes them from the list, so they
get inserted but never replace anything. They accumulate on every single run.
If your partition column can be null, filter those rows out in your model.

## The duplicates that appeared from nowhere

**Symptom.** Row counts climbing faster than they should. `select id, count(*)
... having count(*) > 1` returns plenty.

**Three separate causes.**

**No `unique_key`.** The `MERGE` runs `on FALSE`, nothing ever matches, everything
inserts. If your `is_incremental()` filter ever returns a row you already have,
you now have it twice. Overlapping time windows plus no unique key equals
duplicates on every run.

*Fix:* add a `unique_key`.

**A composite `unique_key` with nullable columns.** This one is genuinely
surprising. When `unique_key` is a list, dbt compares each column with plain `=`:

```sql
on (DBT_INTERNAL_SOURCE.order_id = DBT_INTERNAL_DEST.order_id)
   and (DBT_INTERNAL_SOURCE.line_no = DBT_INTERNAL_DEST.line_no)
```

In SQL, `NULL = NULL` is not true — it's `NULL`, which isn't a match. So any row
with a null in **any** key column never matches, and gets inserted again on every
run.

There's a dbt setting (`enable_truthy_nulls_equals_macro`) that makes comparisons
null-safe, and it genuinely works — **but only for a single-column
`unique_key`.** The list branch never uses it. Turning the flag on does not fix
composite keys.

*Fix:* make sure key columns are never null. `coalesce(line_no, -1)` in your
model is ugly and reliable.

**An `incremental_predicates` window that's too narrow.** If you bound matching
to 7 days and a 30-day-old row changes, it can't match anything inside the
window, so it's inserted as a duplicate.

*Fix:* widen the window to match how far back your data can actually change.

## The column that never appeared

**Symptom.** You added a column, the run succeeded, the column isn't there.

**Cause.** `on_schema_change` defaults to `ignore`. dbt takes the column list
from the **existing table**, so a column that isn't there yet simply isn't in the
insert list. No error, because from dbt's point of view nothing went wrong.

**Fix.** Set it:

```sql
{{ config(
    materialized='incremental',
    on_schema_change='append_new_columns'
) }}
```

Or `dbt run --full-refresh` once, which rebuilds the table with the new shape.

**Also know:** this changes your execution plan, from one statement to two. See
[page 5](05-the-three-strategies.md#the-hidden-setting-that-changes-everything).

## The full refresh that dropped the table

**Symptom.** You changed `partition_by` (or `cluster_by`), ran
`--full-refresh`, and the table was dropped and recreated rather than replaced.
Anything attached to the old table is gone; queries running at that moment
failed.

**Cause.** BigQuery won't let you replace a table with one that has a different
partitioning spec. dbt checks whether the existing table matches your config, and
if not, logs `Hard refreshing <table> because it is not replaceable` and issues a
`DROP` before the `CREATE`.

**Fix.** Nothing to fix — it's the only thing dbt can do. But *expect* it:
changing partitioning is a destructive operation. Plan for the gap, and check
whether anything depends on the table existing continuously. dbt does reapply
grants afterwards.

**Related.** Switching a model from `view` to `incremental` has the same shape.
There's no atomic view-to-table swap on BigQuery, so dbt drops the view first.
Between the drop and the create, the relation doesn't exist.

## The model that costs a fortune

**Symptom.** Your incremental model isn't much cheaper than the full rebuild it
replaced.

**Work through these in order.**

**Is it `merge` on an unpartitioned table?** Then every run scans the whole table
looking for matches. Add partitioning and `incremental_predicates`.

**Are your predicates prunable?** A partition column wrapped in a function kills
pruning. `where cast(event_date as string) = '2026-08-31'` reads everything. Keep
the column bare — [page 4](04-partitioning-explained.md#what-your-query-prunes-actually-requires).

**Did you set `require_partition_filter` and expect it to help?** It doesn't. dbt
satisfies it with a tautology that restricts nothing. You still need a real
bound.

**Is `is_incremental()` actually doing anything?** Read
`target/compiled/` and check the filter is present. A misplaced `{% if %}` can
silently produce a full scan every run.

**Are you reprocessing more than you need?** A 7-day overlap window that could be
2 days costs 3.5× more than necessary, forever.

## The error messages, decoded

These fail at compile time — before any SQL runs, before you're billed. The good
case.

| Message | What it means |
| --- | --- |
| *Invalid incremental strategy provided* | Typo. Only `merge`, `insert_overwrite`, `microbatch`. |
| *The 'insert_overwrite' strategy requires the `partition_by` config* | Add `partition_by`, or switch to `merge`. |
| *The 'microbatch' strategy requires a `partition_by` config* | Same. |
| *requires a `partition_by` config with the same granularity as its configured `batch_size`* | Your `batch_size` and `granularity` disagree. Make them identical. |
| *The 'copy_partitions' option requires ... 'insert_overwrite' or 'microbatch'* | `copy_partitions` can't be used with `merge`. |
| *Model cannot specify merge_update_columns and merge_exclude_columns* | Pick one list, not both. |
| *The 'insert_overwrite' strategy is not yet supported for python models* | Use `merge` for Python models. |
| *The source and target schemas on this incremental model are out of sync!* | `on_schema_change='fail'` did its job. The message lists the differences. |

One runtime error worth naming: **`Undeclared query parameter`** mentioning
`_dbt_max_partition`. That variable only gets declared if the literal text
`_dbt_max_partition` appears in your compiled model. Build the name dynamically
and dbt never declares it. Write it out in full.

## A debugging routine

When an incremental model misbehaves, in this order:

1. **Read the compiled SQL** in `target/compiled/`. Most bugs are visible
   immediately — a missing filter, a predicate that can't prune.
2. **Check `is_incremental()` is doing what you think.** Compare a first run
   against a subsequent one.
3. **Run the model's `select` on its own** and look at what it produces. Empty
   partitions? Nulls in the key?
4. **Compare against a full refresh** in a scratch dataset. If full-refresh
   output differs from incremental output, you've found drift — and the
   difference tells you which failure mode above you're in.
5. **Read the run log**, especially the schema-change lines listing columns
   added and removed.

Step 4 is the one people skip and the one that actually finds the bug. Consider
running it on a schedule.

---

Previous: [5. The three strategies](05-the-three-strategies.md) ·
Next: [7. Where to go next](07-where-to-go-next.md)
