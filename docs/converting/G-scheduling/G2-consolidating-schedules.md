# G2 · Consolidating many schedules into one run

> **Part G — Scheduling, parameters, backfills** · Sourcing: `CRAFT`
> **The question:** forty scripts on forty schedules. How many dbt runs?

Far fewer. Group by **cadence**, not by subject area — everything that runs daily
is one invocation, regardless of what it's about.

## Tag by cadence

```sql
{{ config(tags=['daily']) }}
```

```yaml
models:
  my_project:
    marts:
      +tags: ['daily']
    realtime:
      +tags: ['hourly']
```

```cron
0  2 * * *  dbt build --select tag:daily
0  * * * *  dbt build --select tag:hourly
```

Two entries instead of forty, and adding a model means tagging it rather than
editing cron.

## Why cadence beats subject

Grouping by team or domain feels natural and fights the DAG. A daily marts model
reading an hourly staging model doesn't care which team owns either — it needs the
staging model built first, and `+` handles that.

```bash
dbt build --select tag:daily+
```

Cadence is a property of the schedule. Subject is a property of the code. Only one
of them belongs in cron.

## Watch for models pulled into the wrong cadence

`--select tag:daily` runs only tagged models. `--select +tag:daily` runs their
ancestors too — including hourly ones, which then run twice.

Usually harmless (they're idempotent, or should be —
[E7](../E-translation/E7-idempotency-meaning.md)), but it costs money and can
surprise you. Decide deliberately which form you want, and know that the `+`
prefix crosses cadence boundaries.

## The consolidation is often the biggest win

Forty scripts with staggered offsets means forty guesses about runtime. Consolidate
and you get:

- **Real ordering** instead of timing margin
- **Parallelism** — dbt runs independent models concurrently, bounded by `threads`
- **One failure surface** — a single exit code and one `run_results.json`
- **No drift** when someone adds a dependency

Total wall-clock time usually drops, because the margin between jobs was mostly
idle waiting.

## Threads

```yaml
my_project:
  outputs:
    prod:
      threads: 8
```

More threads means more concurrent BigQuery jobs. Raise it until you hit
concurrency limits or slot contention rather than leaving it at the default 1 or
4 — a consolidated run is where this actually matters.

## Keep separate runs when there's a reason

Don't over-consolidate. Legitimate reasons to keep a separate invocation:

- **Different upstream readiness** — a feed that lands at a different time
- **Different failure tolerance** — something that must not be blocked by an
  unrelated failure
- **Genuinely different cadence**
- **An external step in the middle** — [C14](../C-structural/C14-orchestration.md)

The last is the common one. If an export must happen between two groups of
models, that's two invocations with a step between them.

## Do it after converting, not during

Consolidation is a scheduling change. Converting is a logic change. Doing both at
once makes a failure ambiguous — [K11](../K-antipatterns/K11-convert-and-optimise.md).

Convert first, keeping roughly the old schedule shape. Consolidate once parity is
proven.

---

Previous: [G1 · From cron to `dbt build`](G1-cron-to-dbt-build.md) ·
Next: [G3 · Passing dates in](G3-passing-dates.md)
