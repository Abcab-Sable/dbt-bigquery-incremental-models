# A2 · Map declared dependencies against actual ones

> **Part A — Assess before you convert** · Sourcing: `CRAFT`
> **The question:** what does this script really depend on?

The declared order — the Airflow DAG, the schedule offsets, the wiki diagram — is
what someone believed. The actual dependencies are in the SQL. They differ more
often than not, and the gap is where conversions break.

## Get the real ones from the SQL

Every table a script reads is a dependency, whether or not anything declares it:

```bash
grep -ioE '`?[a-z0-9_-]+`?\.`?[a-z0-9_]+`?\.`?[a-z0-9_]+`?' script.sql | sort -u
```

Crude, and it will catch strings that aren't tables. Good enough to start.

For the authoritative version, ask BigQuery what the job actually read:

```sql
select
    referenced_tables,
    destination_table,
    creation_time
from `region-eu`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
where creation_time > timestamp_sub(current_timestamp(), interval 7 day)
  and destination_table.table_id = 'daily_events'
order by creation_time desc
limit 5;
```

`referenced_tables` is ground truth. It catches what `grep` misses — wildcards,
dynamic SQL, views that read other tables.

## The four gaps to look for

**Undeclared dependency.** The script reads a table the schedule doesn't wait
for. It works because the timing usually holds. Under dbt it becomes a real edge
and the ordering gets enforced — which is an improvement, but may change when
things run.

**Declared but unused.** The DAG waits on something the script never reads.
Free speed once you convert; just confirm it's genuinely unused rather than read
via a view.

**Ordering by schedule offset.** "B runs at 03:15 because A finishes by 03:10."
That isn't a dependency, it's a hope. `ref()` replaces it with an actual edge —
one of the clearest wins available.

**Circular dependencies.** A reads B, B reads A, resolved in practice by
timing. dbt will not permit this. `link_graph` calls `find_cycles()` and raises
`CompilationError: Found a cycle: ...` at parse time.

That last one needs designing out, not converting. Usually it's two concerns
tangled in one table, and separating them is the fix.

## Reading through views

A script reading a view depends on everything that view reads. `referenced_tables`
resolves this for you; `grep` does not. Views are the most common source of
"where did that edge come from" during conversion.

## Record it

Extend the [A1](A1-inventory.md) row:

```
daily_events.sql
  reads:     raw.events, ref.dim_users, analytics.daily_events (self, watermark)
  writes:    analytics.daily_events
  declared:  waits on raw_events_load only
  gap:       ref.dim_users is undeclared — currently works on timing
  cycle:     none
```

The `gap` line is what changes your conversion order in
[I1](../I-migration/I1-conversion-order.md). The self-read is normal — that's
the watermark pattern, and it becomes `{{ this }}`.

---

Previous: [A1 · Inventory your scripts](A1-inventory.md) ·
Next: [A4 · Classify by trigger](A4-classify-by-trigger.md)
