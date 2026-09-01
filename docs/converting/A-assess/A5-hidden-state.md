# A5 · Find the hidden state

> **Part A — Assess before you convert** · Sourcing: `CRAFT`
> **The question:** what does this script assume already exists?

Scripts accumulate assumptions. The table was created by hand three years ago;
someone tops up a lookup each quarter; a column gets patched after a bad load.
None of it is in the SQL. All of it breaks when dbt takes ownership.

## Why it matters more than it sounds

dbt **creates the table**. When you convert, the table's existence, schema,
partitioning and clustering all become products of your model. Anything that was
true because a human made it true stops being true.

The specific failure: `--full-refresh` drops and recreates. Every manual patch,
every hand-added column, every out-of-band correction is gone, and it goes
quietly because dbt is doing exactly what you asked.

## Where to look

**How was the target table created?** If the script only ever `INSERT`s or
`MERGE`s, something else made the table.

```sql
select ddl
from `project.analytics.INFORMATION_SCHEMA.TABLES`
where table_name = 'daily_events';
```

Compare that DDL to what your model config would generate. Differences in
partitioning, clustering, expiration, description or labels are hidden state.

**Are there columns the script never writes?** Diff the table's columns against
the script's insert list. Extras came from somewhere, and under
`on_schema_change: ignore` they'll be silently dropped from the new table.

```sql
select column_name, data_type
from `project.analytics.INFORMATION_SCHEMA.COLUMNS`
where table_name = 'daily_events'
order by ordinal_position;
```

**Who else writes to this table?** The most dangerous case: two producers, one
of which nobody remembers.

```sql
select user_email, statement_type, count(*) as n
from `region-eu`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
where creation_time > timestamp_sub(current_timestamp(), interval 90 day)
  and destination_table.table_id = 'daily_events'
group by 1, 2
order by n desc;
```

More than one principal, or a statement type the script doesn't contain, means
you have a second writer. Find it before you convert, because dbt will happily
overwrite whatever it produces.

**What's manual?** Ask directly, and ask specifically:

- Does anyone re-run this by hand? When, and why?
- Is there a spreadsheet, lookup, or mapping someone maintains?
- Has anyone ever patched the output after a bad run?
- Are there dates known to be wrong that were never fixed?

**Grants and access.** Permissions granted by hand won't survive a drop and
recreate. dbt reapplies the `grants` config — not what someone clicked in the
console two years ago.

```sql
select * from `project.analytics.INFORMATION_SCHEMA.OBJECT_PRIVILEGES`
where object_name = 'daily_events';
```

## What to do with what you find

| Found | Do |
| --- | --- |
| Table DDL differs from your config | Bring it into config, deliberately — [D6](../D-data-movement/D6-partitioning-ddl.md) |
| Columns nobody writes | Decide: model them, or accept losing them |
| A second writer | Resolve before converting. Two producers, one table is a bug either way |
| A manual lookup | Becomes a seed, or a source. Don't leave it manual |
| Hand-granted access | Into the `grants` config — [D10](../D-data-movement/D10-grants-authorized-views.md) |
| Known-bad historical data | Record it in [A9](A9-correctness-baseline.md) so it doesn't look like a conversion bug |

## The question that finds most of it

> **If I dropped this table right now and let the script rebuild it, what would
> be different?**

Anything on that list is hidden state, and `--full-refresh` will find it whether
you did or not.

---

Previous: [A4 · Classify by trigger](A4-classify-by-trigger.md) ·
Next: [A6 · Find the compensating hacks](A6-compensating-hacks.md)
