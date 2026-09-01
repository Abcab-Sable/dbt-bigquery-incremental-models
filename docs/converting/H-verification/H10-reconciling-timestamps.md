# H10 · Reconciling: timestamp precision and timezones

> **Part H — Proving correctness** · Sourcing: `CRAFT`
> **The question:** everything is off by an hour. Where did that come from?

A timezone, nearly always. And if the shift is exactly one partition, it may not
be a display problem — it may mean rows are in the wrong partition entirely.

## The types behave differently

| Type | Stores | Comparison gotcha |
| --- | --- | --- |
| `TIMESTAMP` | An absolute instant, UTC-based | Displayed in session timezone |
| `DATETIME` | Wall-clock, **no** timezone | No conversion happens, ever |
| `DATE` | A calendar date | Derived from whichever of the above, in whichever zone |

Converting between them is where offsets appear. `DATETIME(ts)` uses UTC unless
told otherwise; `DATETIME(ts, 'Europe/London')` doesn't.

## Find the shape of the shift

```sql
select
    countif(old.occurred_at = new.occurred_at)                          as exact,
    countif(timestamp_diff(new.occurred_at, old.occurred_at, hour) = 1) as plus_1h,
    countif(timestamp_diff(new.occurred_at, old.occurred_at, hour) = -1) as minus_1h,
    countif(abs(timestamp_diff(new.occurred_at, old.occurred_at, microsecond)) between 1 and 999999) as sub_second,
    count(*) as compared
from analytics.events old
join analytics_dbt.events new using (event_id);
```

| Result | Cause |
| --- | --- |
| Uniform ±1h | A timezone conversion added or lost |
| ±1h on **some** rows only | **Daylight saving.** A fixed offset was used where a named zone belongs |
| Sub-second differences | Precision — truncation or a cast |
| Whole-day shifts | A `DATE()` extraction in a different zone |

The second row is the important one. A uniform shift is a simple bug. A shift that
applies only to summer dates means someone hardcoded `+01:00` instead of using
`Europe/London`, and it will be wrong twice a year forever.

## The partition consequence

This is where a timezone bug stops being cosmetic.

```sql
select date(occurred_at) as event_date          -- UTC
select date(occurred_at, 'Europe/London') as event_date   -- local
```

For an event at 23:30 UTC on 31 August, those give **different dates**. If
`event_date` is your partition column, the row lands in a different partition —
and with `insert_overwrite`, a different partition gets overwritten.

So a timezone difference can produce:

- rows in the wrong partition
- a partition that looks short while its neighbour looks long
- an `insert_overwrite` clearing the wrong day

Check the boundary directly:

```sql
select
    date(occurred_at) as utc_date,
    date(occurred_at, 'Europe/London') as local_date,
    count(*) as n
from analytics_dbt.events
where date(occurred_at) != date(occurred_at, 'Europe/London')
group by 1, 2
order by 1 desc
limit 10;
```

Non-zero counts mean the choice of zone materially changes your partitioning.
Match whatever the script did, and record it.

## Precision

BigQuery `TIMESTAMP` is microsecond precision. Differences arise from:

- `CURRENT_TIMESTAMP()` captured at different moments — non-deterministic, exclude
  it from comparison ([H3](H3-checksum-parity.md))
- A cast through `DATETIME` or a string losing sub-second digits
- `TIMESTAMP_TRUNC` applied in one implementation and not the other

Sub-second differences on a column that should be copied straight through mean a
cast crept in.

## The run-time offset

Your script ran at 03:00. Your dbt run may not. If either uses
`CURRENT_DATE()` to decide what to process, they process different things — and
the most recent partition will differ for entirely legitimate reasons.

This is why [H5](H5-shadow-mode.md) insists both run minutes apart, and why the
newest partition is usually excluded from parity checks:

```
Excluded from comparison: the current partition (both still receiving data)
```

## Record the zone

```
Timezone: script used DATE(occurred_at) — UTC. Model matches.
          Note: business day is Europe/London; UTC partitioning is
          pre-existing and out of scope for this conversion (DATA-2402).
```

That last line stops someone "fixing" it during the conversion and producing a
diff nobody can explain.

---

Previous: [H9 · Reconciling: numeric precision](H9-reconciling-numeric-precision.md) ·
Next: [H11 · Differences that should exist](H11-differences-that-should-exist.md)
