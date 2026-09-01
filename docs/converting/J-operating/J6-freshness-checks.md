# J6 · Freshness checks

> **Part J — Operating it afterwards** · Sourcing: `CORE✓`
> **The question:** how do we know upstream stopped arriving?

`dbt source freshness`. It's the check that distinguishes "our model is broken"
from "there was nothing to process", and after a conversion that distinction
matters more than it did.

## The config

```yaml
sources:
  - name: raw
    schema: raw
    tables:
      - name: events
        loaded_at_field: _loaded_at
        freshness:
          warn_after: {count: 2, period: hour}
          error_after: {count: 6, period: hour}
```

```bash
dbt source freshness
```

`FreshnessStatus` in dbt-core is `pass · warn · error · runtime error`, matching
the thresholds you set plus a category for the check itself failing.

## Why it matters more after conversion

A script that read stale data produced stale output and failed silently. An
incremental model does the same — it processes whatever is there, succeeds, and
writes nothing new.

So a stalled upstream feed looks exactly like a quiet day. Freshness is what tells
them apart, and without it you'll debug your model when the problem is two systems
upstream.

This is also why declaring sources is worth doing **even for scripts you haven't
converted** — [I2](../I-migration/I2-strangler-pattern.md). You get this check for
free on the whole legacy suite.

## Run it before the build

```bash
dbt source freshness && dbt build --select tag:daily
```

Failing fast on stale sources avoids a run that succeeds while producing nothing —
and avoids the `insert_overwrite` case where an empty batch is
[genuinely dangerous](../B-write-patterns/B14-when-the-range-can-empty.md).

Or select on it:

```bash
dbt build --select source_status:fresher+ --state ./prev
```

Builds only what has new upstream data — useful when sources land irregularly.

## Setting thresholds

From observed behaviour, not aspiration:

```sql
select
    date(_loaded_at) as d,
    min(_loaded_at) as first_load,
    max(_loaded_at) as last_load,
    count(*) as n
from `raw.events`
where _loaded_at > timestamp_sub(current_timestamp(), interval 30 day)
group by d
order by d desc;
```

Set `warn_after` a little beyond the normal worst case and `error_after` at the
point where you'd genuinely want someone to act. Thresholds that fire on ordinary
variation get muted, and a muted check is worse than none.

Remember weekends and holidays — a feed that legitimately pauses will error every
Sunday unless the threshold accounts for it.

## When there's no `loaded_at_field`

Some tables have no load timestamp. Options:

- **Metadata-based freshness**, using the table's last-modified time. dbt supports
  this when `loaded_at_field` is omitted, though it reflects *any* write, not
  arrival of new rows.
- **Add a load timestamp** upstream. Best if you control the loader.
- **A custom test** on the maximum business date, which is often what you actually
  care about:

```sql
-- tests/events_has_recent_data.sql
select 1
from (select max(event_date) as m from {{ source('raw','events') }})
where m < date_sub(current_date(), interval 1 day)
```

## Freshness on your own outputs

Freshness is for sources, but the same idea applies to models — a model that
stopped writing is worth knowing about. That's a recency test
([J2](J2-monitoring-drift.md)) rather than `source freshness`, but it's the same
signal and belongs in the same alert path.

---

Previous: [J5 · Ownership and handover](J5-ownership-handover.md) ·
Next: [J7 · Schema evolution over time](J7-schema-evolution.md)
