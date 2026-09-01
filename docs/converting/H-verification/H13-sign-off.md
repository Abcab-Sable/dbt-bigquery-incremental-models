# H13 · Sign-off, and deleting the old script

> **Part H — Proving correctness** · Sourcing: `CRAFT`
> **The question:** when is this conversion actually finished?

When the criteria you set in [H1](H1-what-correct-means.md) are met, a person has
said so, and the old script is retired rather than merely stopped.

## The checklist

```
Conversion sign-off — analytics.daily_events

Bar (from H1)
  [ ] Column-level parity over 2026-08-01..2026-08-31, all columns
      except _loaded_at                                          → H4
  [ ] Goal was EQUIVALENCE; compensating hacks ported as-is

Evidence
  [ ] Row-count parity, total and per-partition                  → H2
  [ ] Set/fingerprint parity per partition                       → H3
  [ ] Every difference predicted and quantified                  → H11
  [ ] Shadow period criteria met (cases, not days)               → H6

Behaviour
  [ ] Two-run idempotency check passes                           → E8
  [ ] Forced empty-partition test gives the intended result      → B14
  [ ] Late-arriving row picked up, if applicable                 → G8

Standing checks
  [ ] unique + not_null on the unique_key                        → H12
  [ ] Script's implicit guarantees encoded as tests              → H12
  [ ] Model runs under `dbt build`, not `run` then `test`        → H12

Operational
  [ ] Grants in the `grants` config, verified against live       → F11
  [ ] Partitioning, clustering, options match or intentionally differ → A5
  [ ] Downstream consumers notified of agreed differences        → H11
  [ ] Schedule replaces the old one; no double-running           → G1

Sign-off
  [ ] Named owner has accepted:  ________________  date: ________
```

## The empty-partition box

Worth calling out because it's the one most often ticked from memory.

**Actually force it.** Make the model produce zero rows for a partition that has
data, run it, and query that partition. That five-minute test is the only thing
separating a working conversion from
[B14](../B-write-patterns/B14-when-the-range-can-empty.md), and reasoning about
it is not the same as running it.

## Retiring the script properly

Stopping it isn't retiring it. In order:

**1. Disable, don't delete.** Turn off the schedule. Leave the script in place and
runnable for the rollback window — [I6](../BACKLOG.md#part-i--migration-strategy).

**2. Confirm it's actually stopped.** Two writers to one table is worse than
either alone:

```sql
select user_email, statement_type, max(creation_time) as last_run
from `region-eu`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
where creation_time > timestamp_sub(current_timestamp(), interval 7 day)
  and destination_table.table_id = 'daily_events'
group by 1, 2;
```

Only the dbt service account should appear. Anything else is still running.

**3. Wait out the rollback window.** Long enough to cover a monthly close, a
reporting cycle, whatever your calendar demands.

**4. Then delete**, and take the baseline artefacts with it —
[I8](../BACKLOG.md#part-i--migration-strategy). Keep the baseline document; it
explains why the model looks the way it does.

**5. Remove the source declaration.** If downstream models pointed at the script's
output via `source()`, they should now `ref()` the model. Nothing should reference
both — [E3](../E-translation/E3-ref-vs-source.md):

```bash
grep -rn "source('legacy', 'daily_events')" models/
```

Zero results, or the conversion is half-done.

## What "signed off" needs to mean

A named person, not a team. Someone who can say what correct output looks like —
the owner you identified in [A1](../A-assess/A1-inventory.md).

If you can't find that person, you cannot sign off, because there is nobody to
accept the differences in [H11](H11-differences-that-should-exist.md). That was
the warning in [A7](../A-assess/A7-what-not-to-convert.md), arriving late.

## Then it isn't finished

Sign-off ends the conversion. It doesn't end the risk — every silent failure in
this documentation shows up *after* cutover, not during it.

The standing control is a scheduled full-refresh reconciliation:
[J3](../BACKLOG.md#part-j--operating-it-afterwards). Set it up as part of the
conversion, while you still remember what correct looked like.

---

Previous: [H12 · Tests from the script's guarantees](H12-tests-from-guarantees.md) ·
**End of wave 3.** Back to [the backlog](../BACKLOG.md#delivery-waves)
