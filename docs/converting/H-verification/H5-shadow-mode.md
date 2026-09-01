# H5 · Shadow mode: running both in parallel

> **Part H — Proving correctness** · Sourcing: `CRAFT`
> **The question:** how do I prove it matches over time, not just once?

Run both, side by side, on the same schedule, writing to different tables. Compare
daily. It is the only technique that catches behaviour rather than a snapshot.

## The setup

Old script keeps running, untouched, writing to `analytics.daily_events`.

New model runs on the same cadence into a parallel dataset:

```yaml
# dbt_project.yml
models:
  my_project:
    +schema: "{{ 'shadow' if target.name == 'shadow' else none }}"
```

```bash
dbt build --select daily_events --target shadow
```

Nothing downstream points at the new table. Nothing is at risk. You accumulate
evidence.

## Why a snapshot isn't enough

A one-off comparison runs both against the same input at the same moment. That
misses everything time-dependent:

- **Late-arriving data** — arrives between runs, and the two may handle it
  differently
- **Empty periods** — the [B14](../B-write-patterns/B14-when-the-range-can-empty.md)
  case, which needs a quiet day to occur naturally
- **Boundary timing** — a run at 23:58 vs 00:02 lands in different partitions
- **Upstream mutation** — a source row changing after both have read it
- **Accumulating drift** — the failure mode that is invisible on day one and
  obvious on day thirty

That last one is the whole argument. Every silent failure in this documentation
looks fine on the first run.

## Same schedule, same inputs

The comparison is only meaningful if both see the same world:

- **Same time.** Run them minutes apart, not hours. A twelve-hour gap makes every
  recent partition differ for uninteresting reasons.
- **Same upstream.** If the old script reads a table another job rebuilds, both
  must read the same version.
- **Same parameters.** Same lookback window, same date bounds.

Where you can't align them, note it in [H1](H1-what-correct-means.md) so the
resulting differences aren't mistaken for bugs.

## Compare automatically

Manual comparison stops after three days. Make it a scheduled query writing to a
results table:

```sql
insert into ops.shadow_parity
with old as (
    select event_date, count(*) n, sum(farm_fingerprint(to_json_string(t))) fp
    from analytics.daily_events t group by event_date
),
new as (
    select event_date, count(*) n, sum(farm_fingerprint(to_json_string(t))) fp
    from analytics_shadow.daily_events t group by event_date
)
select
    current_timestamp() as checked_at,
    coalesce(old.event_date, new.event_date) as event_date,
    old.n as old_n, new.n as new_n,
    old.fp is not distinct from new.fp as matches
from old full outer join new using (event_date)
where old.n is distinct from new.n or old.fp is distinct from new.fp;
```

An empty insert is a clean day. The table becomes your evidence trail for
[H13](H13-sign-off.md), and a row appearing after four clean days is exactly the
signal shadow mode exists to produce.

## Cost

You are running everything twice. For most models that's noise; for an expensive
one it isn't.

If cost is prohibitive:

- Shadow a **representative subset** — a few partitions, or one tenant
- Shadow for a **shorter window** but include a known edge case deliberately
- Shadow **weekly** rather than daily, accepting weaker evidence

Any of these beats no shadow period. Skipping it entirely is a decision to find
out in production.

## Force the edge cases

Don't wait for a quiet day to occur. During the shadow period, deliberately:

- Make a partition produce zero rows and confirm both behave the same —
  [B14](../B-write-patterns/B14-when-the-range-can-empty.md)
- Run the new model twice and confirm no duplication —
  [E8](../E-translation/E8-idempotency-proving.md)
- Replay a late-arriving row and check both pick it up

Shadow mode gives you a safe place to do this. Use it, rather than only watching.

---

Previous: [H4 · Column-level diffing](H4-column-level-diffing.md) ·
Next: [H6 · How long to shadow](H6-shadow-duration.md)
