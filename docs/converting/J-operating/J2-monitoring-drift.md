# J2 · Monitoring incremental drift in production

> **Part J — Operating it afterwards** · Sourcing: `CRAFT`
> **The question:** how would we know if the model quietly stopped being correct?

You wouldn't, unless you check. That's the defining property of every failure in
this documentation — the run succeeds, the counts look plausible, and the data is
wrong.

## What drift is

An incremental model's output diverges from what a full rebuild would produce.
Causes, all documented elsewhere and all silent:

- A partition that should be empty keeps its old rows —
  [B14](../B-write-patterns/B14-when-the-range-can-empty.md)
- Duplicates from a nullable composite key —
  [B8](../B-write-patterns/B8-merge-on-clause-to-unique-key.md)
- A new column dropped under `on_schema_change: ignore` —
  [D8](../D-data-movement/D8-add-column-migrations.md)
- Rows older than `incremental_predicates` inserted as duplicates —
  [B12](../B-write-patterns/B12-extra-predicates.md)
- Late data arriving outside the lookback window —
  [G8](../G-scheduling/G8-late-arriving-data.md)

None of these fail a run. All of them accumulate.

## The only reliable detector

Rebuild in a scratch dataset and diff against production. That's
[J3](J3-scheduled-reconciliation.md), and it's the control that actually works.

Everything below is cheaper and weaker — useful for catching drift early between
reconciliations.

## Cheap standing checks

**Uniqueness on the key**, as a test:

```yaml
models:
  - name: orders
    columns:
      - name: order_id
        data_tests: [unique, not_null]
```

Catches duplicate accumulation on the next run rather than the fiftieth. If you
add nothing else, add this.

**Volume in a plausible band:**

```sql
-- tests/daily_events_plausible_volume.sql
select event_date, count(*) as n
from {{ ref('daily_events') }}
where event_date >= date_sub(current_date(), interval 7 day)
group by event_date
having count(*) < 100 or count(*) > 10000000
```

Crude, and it catches an empty run or a runaway join before anyone notices the
dashboard.

**Recency:**

```sql
-- tests/daily_events_has_yesterday.sql
select 1
from (select max(event_date) as m from {{ ref('daily_events') }})
where m < date_sub(current_date(), interval 1 day)
```

Catches a model that silently stopped writing — which a green run will not tell
you, because the model succeeded at producing nothing.

**Row-count trend**, as a warn-severity check. A sudden step change in daily
volume is worth a look even when it's within the plausible band.

## What green runs don't mean

Worth stating explicitly for whoever inherits this:

> A successful dbt run means the SQL executed. It does not mean the data is
> correct. Most incremental failure modes produce successful runs.

Put that in the runbook ([I10](../I-migration/I10-documenting-decisions.md)). The
instinct to treat a green pipeline as evidence is strong and wrong.

## Watch the shape, not just the totals

Drift usually shows per-partition before it shows in a total:

```sql
select event_date, count(*) as n
from analytics.daily_events
where event_date >= date_sub(current_date(), interval 30 day)
group by event_date
order by event_date;
```

A day that doesn't move when its neighbours do — or one that's been identical for
three weeks while upstream changed — is the empty-partition signature.

## Alert on the right things

Tests failing should page someone; see [J4](J4-alerting.md) for what deserves
which severity. Drift detected by reconciliation should open a ticket, not wake
anyone — by definition it's been wrong for a while already.

---

Previous: [J1 · Cost after conversion](J1-cost-after-conversion.md) ·
Next: [J3 · Scheduled full-refresh reconciliation](J3-scheduled-reconciliation.md)
