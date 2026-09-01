# A9 · Capture the correctness baseline

> **Part A — Assess before you convert** · Sourcing: `CRAFT`
> **The question:** what do I record before I touch anything?

Once the script stops running, you can't ask it what it used to do. Everything in
[Part H](../BACKLOG.md#part-h--proving-correctness) compares the new thing against
the old thing — so the old thing has to be written down first.

This is fifteen minutes of work that is impossible to do retroactively.

## Snapshot the output

Copy the table before you touch anything. Not a count — the actual rows.

```sql
create table analytics_baseline.daily_events_20260901 as
select * from analytics.daily_events;
```

Cheap, and it's the only artefact that lets you diff column values later. If the
table is too large, snapshot a representative slice — a month, or a stratified
sample across partitions — and record which slice you took.

BigQuery's time travel gives you seven days by default, which is often not long
enough for a conversion. Make the copy.

## Record the shape

```sql
select
    count(*)                        as total_rows,
    count(distinct event_date)      as distinct_partitions,
    min(event_date)                 as earliest,
    max(event_date)                 as latest,
    countif(event_date is null)     as null_partition_rows
from analytics.daily_events;
```

Then per partition, since that's the resolution the failures happen at:

```sql
create table analytics_baseline.daily_events_counts_20260901 as
select event_date, count(*) as n
from analytics.daily_events
group by event_date;
```

That second table is what [H2](../H-verification/H2-row-count-parity.md) joins
against. Having it precomputed makes parity checking a single query for the rest
of the conversion.

Also capture the schema, so [D8](../BACKLOG.md#part-d--data-movement-ddl-and-metadata)
has something to compare against:

```sql
select column_name, data_type, is_nullable
from `project.analytics.INFORMATION_SCHEMA.COLUMNS`
where table_name = 'daily_events'
order by ordinal_position;
```

## Record the behaviour, not just the data

This is the part people skip, and it's the part that matters most.

**How often does it run, and at what time?** The conversion has to reproduce the
cadence, and "3am UTC" versus "3am local" changes which partition is current.

**What range does each run touch?** Read it out of the script — the `DELETE`
bounds, the watermark, the parameter. This becomes your `is_incremental()` filter
and, if needed, your `partitions` list.

**What does it do when a period has no data?** The question from
[A3](A3-classify-by-write-pattern.md), and the one that decides whether your
[B13](../B-write-patterns/B13-delete-insert-to-insert-overwrite.md) conversion is
correct. Read the script and answer explicitly:

> `DELETE FROM daily_events WHERE event_date >= start_date` — unconditional.
> An empty day **is emptied**. Conversion must preserve this ⇒ static `partitions`.

**What does it do when re-run?** Run it twice in a scratch dataset if you safely
can, and record whether the output changes. If the old script wasn't idempotent,
you need to know that before you conclude the new one is broken.

**How long does it take, and what does it cost?** From
`INFORMATION_SCHEMA.JOBS`:

```sql
select
    creation_time,
    total_bytes_processed,
    total_slot_ms,
    timestamp_diff(end_time, start_time, second) as duration_s
from `region-eu`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
where statement_type != 'SCRIPT'
  and query like '%daily_events%'
  and creation_time > timestamp_sub(current_timestamp(), interval 30 day)
order by creation_time desc
limit 20;
```

Without this you can't claim the conversion improved anything, and "it feels
faster" doesn't survive a review.

## Record the edge cases

Ask whoever owns it:

- Has it ever produced wrong output? What did that look like?
- Is there a manual step anyone performs around it?
- Has anyone ever re-run it by hand, and why?
- Are there dates known to be wrong that nobody has fixed?

That last one saves real time. Converting a script whose output is already wrong
for three days in 2024 produces a diff you'll spend a day chasing. Knowing about
it upfront turns a bug hunt into a footnote — and connects to
[A6](../BACKLOG.md#part-a--assess-before-you-convert), where compensating hacks
get found.

## Where to put it

One file per script, next to the model you're about to write:

```
models/events/daily_events.baseline.md
```

Containing: snapshot table names and date, the shape numbers, the schedule and
range, the empty-period answer, the re-run answer, cost figures, and the
edge-case notes.

Commit it. It's the evidence for the sign-off in
[H13](../BACKLOG.md#part-h--proving-correctness), and the thing that tells the
next person why the model is shaped the way it is.

## The trap

**Capturing only the happy path.** A baseline taken on a normal Tuesday tells you
what normal looks like and nothing about what breaks. The three questions that
actually predict conversion bugs are the behavioural ones — empty periods,
re-runs, known-bad data — and none of them show up in a row count.

If you're short on time, skip the cost figures and answer those three.

---

Previous: [H2 · Row-count parity](../H-verification/H2-row-count-parity.md) ·
**End of wave 1.** Back to [the backlog](../BACKLOG.md#delivery-waves)
