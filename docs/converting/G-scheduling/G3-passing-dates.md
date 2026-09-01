# G3 · Passing dates in: `vars` and `--vars`

> **Part G — Scheduling, parameters, backfills** · Sourcing: `CORE`
> **The question:** the scheduler passed a date. How does dbt receive it?

As a var. The mechanics are in
[E11](../E-translation/E11-query-parameters.md); this is the scheduling side —
where the value comes from and what happens when it's absent.

## Defaults matter more than you think

```sql
{% set run_date = var('run_date', none) %}

select * from {{ source('raw', 'events') }}
where event_date = {% if run_date %} date('{{ run_date }}') {% else %} current_date() {% endif %}
```

**Always give `var()` a default.** A var with no default and no value raises at
compile time, so a scheduled run that forgets to pass it fails outright.

Whether that's good depends on the var. For a backfill date, failing loudly is
right. For a lookback window, a sensible default is better than a broken schedule.

## Where the value comes from

**Airflow**, templating the execution date:

```python
BashOperator(
    task_id='dbt_build',
    bash_command=(
        'dbt build --select daily_events '
        '--vars \'{run_date: {{ ds }}}\''
    ),
)
```

**Cron**, computing it in the shell:

```bash
dbt build --select daily_events --vars "{run_date: $(date -u -d yesterday +%F)}"
```

**Not at all**, letting the model use `current_date()`. Usually the right answer
for a plain daily run — the fewer moving parts between the scheduler and the
model, the fewer ways the date can be wrong.

## Prefer no parameter where you can

A script needed a parameter because it had no other way to know when it was. A
model can just ask:

```sql
where event_date >= date_sub(current_date(), interval 3 day)
```

Keep the parameter only if someone genuinely overrides it — for backfills, or
replays. Otherwise you've added a coupling between the scheduler and the model
that has to stay correct forever.

## Timezone, again

`{{ ds }}` in Airflow is the logical date in UTC. `date` in a shell is the box's
local time unless you pass `-u`. `current_date()` in BigQuery is UTC unless given
a zone.

Three places to get it wrong, and the symptom is an off-by-one-day partition that
looks like a conversion bug. Pin everything to UTC and be explicit —
[H10](../H-verification/H10-reconciling-timestamps.md).

## Vars are substituted, not bound

Worth repeating from [E11](../E-translation/E11-query-parameters.md): a var is
written into the SQL text at compile time. It is not a bound parameter.

So quote and cast explicitly, and never build a var from untrusted input:

```sql
where event_date = date('{{ var("run_date") }}')
```

## For backfills, prefer microbatch

If the parameter exists so a bash loop can iterate dates, dbt has a better answer:

```bash
dbt run --select daily_events --event-time-start 2024-01-01 --event-time-end 2024-06-30
```

Batch boundaries, ordering and retry handled for you, with the boundary maths
documented and pinned — [the microbatch page](../../balanced/05-microbatch.md).
See [G6](G6-backfill-microbatch.md).

---

Previous: [G2 · Consolidating schedules](G2-consolidating-schedules.md) ·
Next: [G4 · Environment variables and secrets](G4-env-vars-secrets.md)
