# J3 · Scheduled full-refresh reconciliation

> **Part J — Operating it afterwards** · Sourcing: `CRAFT`
> **The question:** what's the one control worth building after cutover?

This one. Periodically rebuild the model from scratch in a scratch dataset and
diff it against production. It is the only reliable detector for every silent
failure in this documentation, and almost nobody does it.

## The idea

An incremental model is an optimisation of "rebuild from source". If the
optimisation is correct, the two agree. If they disagree, the optimisation has
drifted — and you now know, with the diff telling you where.

```bash
dbt run --select daily_events --full-refresh --target recon
```

Then compare `analytics_recon.daily_events` against `analytics.daily_events`.

## The comparison

Reuse the parity queries from Part H — same technique, now as a standing control:

```sql
insert into ops.reconciliation
with prod as (
    select event_date, count(*) n, sum(farm_fingerprint(to_json_string(t))) fp
    from analytics.daily_events t group by event_date
),
recon as (
    select event_date, count(*) n, sum(farm_fingerprint(to_json_string(t))) fp
    from analytics_recon.daily_events t group by event_date
)
select
    current_timestamp() as checked_at,
    'daily_events' as model,
    coalesce(prod.event_date, recon.event_date) as event_date,
    prod.n as prod_rows,
    recon.n as recon_rows,
    coalesce(recon.n, 0) - coalesce(prod.n, 0) as delta
from prod full outer join recon using (event_date)
where prod.n is distinct from recon.n
   or prod.fp is distinct from recon.fp;
```

An empty insert is a clean reconciliation. Rows are your drift, by partition.

Exclude non-deterministic columns and mind float precision —
[H3](../H-verification/H3-checksum-parity.md),
[H9](../H-verification/H9-reconciling-numeric-precision.md).

## How often

Balance cost against how long you're willing to be wrong:

| Model | Cadence |
| --- | --- |
| Feeds finance or external reporting | Weekly |
| Ordinary internal marts | Monthly |
| Large and expensive to rebuild | Quarterly, or a sampled range |
| Uses dynamic `insert_overwrite` | More often — highest drift risk |

If a full rebuild is prohibitive, reconcile a **window** — the last 90 days, or a
stratified sample of partitions. Weaker, still far better than nothing.

## Expect some noise

Some differences are legitimate and recurring:

- **The current partition**, still receiving data. Exclude it.
- **Late-arriving data** that landed between the two builds.
- **Non-deterministic columns** — `_loaded_at`, generated ids.
- **Float summation** — [H9](../H-verification/H9-reconciling-numeric-precision.md).

Codify those exclusions once, so a real difference stands out instead of being
lost in known noise. A reconciliation everyone ignores is worse than none, because
it creates the impression of a control.

## What to do with a finding

The drift tells you which failure mode you have:

| Shape | Likely cause |
| --- | --- |
| Prod has rows in a partition recon doesn't | [Empty-partition trap](../B-write-patterns/B14-when-the-range-can-empty.md) |
| Prod has more rows, scattered | Duplicates — key or predicate window |
| Recon has a column prod lacks | `on_schema_change: ignore` |
| Old partitions differ, recent match | Late data outside the lookback |
| Everything differs | The model changed and prod hasn't been rebuilt |

That last one is common and benign — someone edited the model and only new
partitions reflect it. Worth knowing rather than investigating as a bug.

## Set it up during the conversion

While you still remember what correct looks like, and while the parity queries
from Part H are fresh. Retrofitting it a year later means reconstructing all of
that.

It's also the honest answer to "how will we know if the conversion was wrong" —
better than "we tested it carefully", because it keeps answering.

---

Previous: [J2 · Monitoring incremental drift](J2-monitoring-drift.md) ·
Next: [J4 · Alerting: test failures vs run failures](J4-alerting.md)
