# G7 · Backfill via explicit partition ranges

> **Part G — Scheduling, parameters, backfills** · Sourcing: `SRC`
> **The question:** I want to rebuild March. Not everything, just March.

Drive the `partitions` config from vars and run it for the range you want. More
manual than [microbatch](G6-backfill-microbatch.md), and useful when you want
precise control over exactly which partitions are touched.

## The setup

```sql
{{ config(
    materialized='incremental',
    incremental_strategy='insert_overwrite',
    partition_by={'field': 'event_date', 'data_type': 'date', 'granularity': 'day'},
    partitions=(
        "date_sub(current_date(), interval 2 day)",
        "date_sub(current_date(), interval 1 day)",
        "current_date()"
    ) if not var('backfill_start', false) else dbt.partition_range(
        var('backfill_start'), var('backfill_end')
    )
) }}
```

That gets unwieldy fast. Cleaner with a macro:

```sql
{% macro partitions_to_replace() %}
  {%- if var('backfill_start', none) -%}
    {%- set start = var('backfill_start') -%}
    {%- set end = var('backfill_end') -%}
    {{ return([
        "date('" ~ start ~ "')",
        "date('" ~ end ~ "')"
    ]) }}
  {%- else -%}
    {{ return([
        "date_sub(current_date(), interval 1 day)",
        "current_date()"
    ]) }}
  {%- endif -%}
{% endmacro %}
```

```sql
{{ config(
    materialized='incremental',
    incremental_strategy='insert_overwrite',
    partition_by={'field': 'event_date', 'data_type': 'date', 'granularity': 'day'},
    partitions=partitions_to_replace()
) }}
```

```bash
dbt run --select daily_events --vars '{backfill_start: 2026-03-01, backfill_end: 2026-03-31}'
```

## Keep the model filter in step

The `partitions` list says which partitions get **cleared**. Your
`is_incremental()` filter says which rows get **produced**. If they disagree you
either clear partitions you don't rebuild, or rebuild rows into partitions you
didn't clear.

Derive both from the same vars:

```sql
{% if is_incremental() %}
  where event_date between
      {{ "date('" ~ var('backfill_start') ~ "')" if var('backfill_start', none) else "date_sub(current_date(), interval 1 day)" }}
      and
      {{ "date('" ~ var('backfill_end') ~ "')" if var('backfill_end', none) else "current_date()" }}
{% endif %}
```

Verbose. Worth it — a mismatch here silently deletes data.

## Values are interpolated verbatim

The `partitions` list goes straight into the SQL:

```jinja
{{ partition_by.render_wrapped(alias='dbt_internal_dest') }} in (
    {{ partitions | join(', ') }}
)
```

They're **SQL expressions**, not literals. `current_date()` works; a fixed date
needs its own quotes. There's no escaping and no type checking, so never build
this list from untrusted input.

They must also be at **partition granularity**. With monthly partitioning,
`'2026-03-15'` will never match `date_trunc(dest, month)` —
[the balanced track](../../balanced/06-partition-config.md#where-each-rendering-is-used).

## The advantage over dynamic

Static `partitions` clears the listed partitions **whether or not your model
produced rows for them**. For a backfill that's exactly what you want — a day with
no data should end up empty, not keep whatever was there before.

That's [B14](../B-write-patterns/B14-when-the-range-can-empty.md) working in your
favour, and it's the main reason to prefer this form for backfills.

## Chunk it

Don't backfill three years in one invocation. The temp table holds the whole
range, and the `MERGE` covers every listed partition:

```bash
for m in 01 02 03 04 05 06; do
  dbt run --select daily_events \
    --vars "{backfill_start: 2026-${m}-01, backfill_end: 2026-${m}-28}"
done
```

At which point [microbatch](G6-backfill-microbatch.md) is doing the same thing
with better bookkeeping — so prefer it unless you specifically need the manual
control.

## Verify after

```sql
select event_date, count(*) as n
from analytics.daily_events
where event_date between '2026-03-01' and '2026-03-31'
group by event_date order by event_date;
```

Check every day you intended is present, and that days you didn't intend to touch
are unchanged. Compare against the baseline —
[H2](../H-verification/H2-row-count-parity.md).

---

Previous: [G6 · Backfill via microbatch](G6-backfill-microbatch.md) ·
Next: [G8 · Late-arriving data](G8-late-arriving-data.md)
