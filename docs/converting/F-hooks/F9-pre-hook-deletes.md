# F9 · pre-hook: deleting rows before insert

> **Part F — Hooks** · Sourcing: `CRAFT`
> **The question:** my script deletes then inserts. Can the delete be a pre-hook?

It can. It shouldn't be. This is the single most common bad conversion of the
delete-insert archetype, and it's worth its own page because it looks so
reasonable.

## The tempting version

```sql
{{ config(
    materialized='incremental',
    pre_hook="delete from {{ this }} where event_date >= date_sub(current_date(), interval 3 day)"
) }}

select event_date, user_id, count(*) as event_count
from {{ source('raw', 'events') }}
where event_date >= date_sub(current_date(), interval 3 day)
group by 1, 2
```

It mirrors the script one-for-one. It works. And it's worse than the alternative
in four separate ways.

## Why it's wrong

**1. It isn't atomic.** The delete is one statement, the insert another. Between
them the table is missing three days of data. If the model fails — bad SQL,
permissions, a quota — you are left with **deleted data and no replacement**.

Your original script had this problem too. `insert_overwrite` does not: the
delete and insert happen in a single `MERGE`.

**2. The range is stated twice.** The hook and the model body both carry
`interval 3 day`. Someone will change one and not the other, and the failure is
silent — either a gap or a duplicate, depending which way it drifts.

**3. It's DML on the model's own table from outside the materialization.** dbt
believes it owns this relation. A pre-hook modifying it is invisible to the
materialization's logic, to `on_schema_change`, and to anyone reading the model.

**4. It defeats the point of converting.** You've kept the script's structure and
gained only the file location. That's [K5](../K-antipatterns/K5-imperative-jinja.md).

## The right version

```sql
{{ config(
    materialized='incremental',
    incremental_strategy='insert_overwrite',
    partition_by={'field': 'event_date', 'data_type': 'date', 'granularity': 'day'},
    partitions=[
        'date_sub(current_date(), interval 2 day)',
        'date_sub(current_date(), interval 1 day)',
        'current_date()'
    ]
) }}

select event_date, user_id, count(*) as event_count
from {{ source('raw', 'events') }}
where event_date >= date_sub(current_date(), interval 2 day)
group by 1, 2
```

The delete is gone. dbt generates it as part of the `MERGE`, scoped to those
partitions, atomic with the insert. Full walkthrough in
[B13](../B-write-patterns/B13-delete-insert-to-insert-overwrite.md).

The range is still expressed twice — once in `partitions`, once in the filter —
which is inherent to the static form. Keep them adjacent so the pairing is
obvious, and consider deriving both from one var
([E6](../E-translation/E6-hardcoded-dates.md)).

## The cases people cite

**"My table isn't partitioned."** Then partition it. If the delete filters on a
column, that column is your partition key —
[B13](../B-write-patterns/B13-delete-insert-to-insert-overwrite.md). If it can't
be partitioned, `merge` with a `unique_key` is the answer, not a pre-hook delete.

**"I delete on something that isn't the partition column."** Then it isn't a
delete-insert; it's an upsert or a retention policy wearing a disguise. Upsert ⇒
[B8](../B-write-patterns/B8-merge-on-clause-to-unique-key.md). Retention ⇒ a
separate scheduled operation, not attached to this model.

**"The delete condition is complex."** Complexity is an argument for making it
visible in a model, not for hiding it in a config string.

## If you genuinely must

Sometimes a migration needs the crude version for a week. If so, make the danger
legible:

```sql
{{ config(
    materialized='incremental',
    -- TEMPORARY (DATA-2310): mirrors legacy delete until events is repartitioned.
    -- NOT ATOMIC — a failure here leaves 3 days missing. Do not run unattended.
    pre_hook="delete from {{ this }} where event_date >= date_sub(current_date(), interval 3 day)"
) }}
```

A ticket, an expiry, and an explicit statement of the failure mode. Without those
three, it's permanent.

---

Previous: [F8 · pre-hook patterns worth keeping](F8-pre-hook-patterns.md) ·
Next: [F10 · post-hook patterns worth keeping](F10-post-hook-patterns.md)
