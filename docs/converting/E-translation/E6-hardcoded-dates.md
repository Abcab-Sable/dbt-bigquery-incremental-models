# E6 · Hardcoded dates and backfill parameters

> **Part E — Statement-level translation** · Sourcing: `CRAFT`
> **The question:** my script has a `start_date` at the top. Where does it go?

Almost every script has one — a `DECLARE`, a substituted parameter, or a literal
someone edits when backfilling. It becomes one of three things, and picking the
wrong one is how you end up editing model files to run a backfill.

## The three destinations

**Inline expression** — when the value is always derived the same way:

```sql
{% if is_incremental() %}
  where event_date >= date_sub(current_date(), interval 3 day)
{% endif %}
```

Right for the common case. The script's `DECLARE start_date DEFAULT
DATE_SUB(CURRENT_DATE(), INTERVAL 3 DAY)` had no reason to be a variable except
that SQL scripts need one to reuse a value.

**A var** — when someone needs to override it at run time:

```sql
{% set lookback = var('lookback_days', 3) %}

{% if is_incremental() %}
  where event_date >= date_sub(current_date(), interval {{ lookback }} day)
{% endif %}
```

```bash
dbt run --select daily_events --vars '{lookback_days: 30}'
```

**A `partitions` list** — when the script's range was a declared set of periods
rather than a filter, which is the `insert_overwrite` case:

```sql
partitions=['date_sub(current_date(), interval 1 day)', 'current_date()']
```

See [B13](../B-write-patterns/B13-delete-insert-to-insert-overwrite.md) and
[B14](../B-write-patterns/B14-when-the-range-can-empty.md).

## Which one

| The script's date is | Use |
| --- | --- |
| Always the same relative expression | Inline |
| Edited by hand for backfills | A var |
| Passed in by the scheduler | A var — [G3](../BACKLOG.md#part-g--scheduling-parameters-backfills) |
| A set of periods to replace | `partitions` |
| A fixed historical literal (`'2019-01-01'`) | Inline, and ask why |

## The one to watch: edited-by-hand

If backfilling currently means opening the file, changing a date, running it, and
changing it back — that must become a var. Otherwise you've converted a bad
practice into a bad practice in a version-controlled file, where the temporary
edit can be committed by accident.

The tell in the script is a commented-out line:

```sql
DECLARE start_date DATE DEFAULT DATE_SUB(CURRENT_DATE(), INTERVAL 3 DAY);
-- DECLARE start_date DATE DEFAULT '2024-01-01';  -- backfill
```

That comment is a feature request. It becomes `var('start_date', ...)`.

## Fixed historical literals

```sql
where event_date >= '2019-01-01'
```

Usually one of: the date the data starts, the date a migration happened, or the
date someone decided older data was untrustworthy. Find out which, because it
changes what you do:

- Data genuinely starts there ⇒ keep it, comment it
- A migration boundary ⇒ keep it, comment it, note it in [A9](../A-assess/A9-correctness-baseline.md)
- Someone's judgement about bad data ⇒ that's a compensating hack, [A6](../A-assess/A6-compensating-hacks.md)

## Don't over-parameterise

The opposite failure is turning every literal into a var "for flexibility". You
get a model nobody can read and defaults nobody can find. That's
[K7](../BACKLOG.md#part-k--anti-patterns).

Parameterise what someone actually changes. Everything else is a constant, and
constants are clearer inline.

---

Previous: [E5 · Finding every hardcoded table name](E5-finding-hardcoded-names.md) ·
Next: [E7 · Idempotency: what it means](E7-idempotency-meaning.md)
