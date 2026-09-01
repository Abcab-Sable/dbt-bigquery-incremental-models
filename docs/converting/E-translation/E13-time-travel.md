# E13 · Time travel and `FOR SYSTEM_TIME AS OF`

> **Part E — Statement-level translation** · Sourcing: `CRAFT`
> **The question:** my script reads a table as it was an hour ago. Does that convert?

The syntax converts trivially. The question worth asking is why the script needed
it, because the usual reasons don't apply after conversion.

## The syntax

```sql
SELECT * FROM analytics.orders
FOR SYSTEM_TIME AS OF TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 HOUR);
```

BigQuery keeps seven days of history by default. In a model:

```sql
select * from {{ ref('orders') }}
for system_time as of timestamp_sub(current_timestamp(), interval 1 hour)
```

`ref()` still produces the edge, so lineage is intact.

## Why scripts use it

Three reasons, and two of them stop applying:

**Consistency across statements.** Reading the same snapshot in several statements
so they can't disagree mid-run. After conversion, a model is one statement — the
inconsistency it guarded against can't happen ([E1](E1-one-statement-per-model.md)).

**Avoiding a concurrent writer.** Reading a stable version while something else
writes. If both are now dbt models, `ref()` orders them and the race is gone
([E2](E2-ordering-by-ref.md)).

**Genuinely wanting historical state.** Reconciliation, audit, "what did we think
yesterday". This one is real and stays.

Work out which before porting it. Carrying forward time travel that was
compensating for a race means keeping a workaround for a problem you just fixed —
and it makes the model non-deterministic for no reason.

## It makes parity checking harder

A model reading `AS OF` an hour ago produces different output depending on when it
runs. Two runs minutes apart legitimately differ.

That interacts badly with [H3](../H-verification/H3-checksum-parity.md) and
[H5](../H-verification/H5-shadow-mode.md) — you can't compare old and new unless
both read the same snapshot, which they won't.

If you must keep it, pin the timestamp during comparison:

```sql
{% set as_of = var('as_of', none) %}
select * from {{ ref('orders') }}
{% if as_of %} for system_time as of timestamp('{{ as_of }}') {% endif %}
```

Then both sides can be pointed at the same instant for the duration of
verification.

## The genuinely useful case: recovery

Time travel's best use during a conversion isn't in a model at all — it's undoing
a mistake:

```sql
create or replace table analytics.daily_events_recovered as
select * from analytics.daily_events
for system_time as of timestamp_sub(current_timestamp(), interval 2 hour);
```

If a converted model wrote something wrong, this recovers the prior state — within
the seven-day window.

**Seven days is often not long enough for a conversion**, which is why
[A9](../A-assess/A9-correctness-baseline.md) says to take a real snapshot table
rather than relying on time travel.

## It doesn't survive a recreate

Time travel is per-table-version. If dbt **drops and recreates** the table —
changed partitioning ([D6](../D-data-movement/D6-partitioning-ddl.md)), or a
`view` → `table` change ([B3](../B-write-patterns/B3-create-view.md)) — the
history goes with the old table.

So the recovery route above is unavailable in exactly the situation where you're
most likely to want it. Another reason for a real baseline snapshot before
starting.

## Snapshots, if you want history properly

If the script used time travel to build a slowly-changing dimension or track
changes over time, dbt snapshots are the purpose-built answer — they record change
history in a table you control, with no seven-day limit.

That's a different materialization and out of scope here, but it's the right
destination for "we need to know what this looked like before".

---

Previous: [E12 · Cost controls](E12-cost-controls.md) ·
**End of wave 4.** Back to [the backlog](../BACKLOG.md#delivery-waves)
