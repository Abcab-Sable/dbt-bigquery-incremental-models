# B10 · `WHEN NOT MATCHED BY SOURCE THEN DELETE`

> **Part B — Write-pattern archetypes** · Sourcing: `SRC`
> **The question:** my `MERGE` deletes rows that vanished from source. What's the dbt config?

There isn't one. This is the honest-answer page: dbt's `merge` strategy cannot
express this clause, and no config adds it.

## What your script does

```sql
MERGE INTO analytics.orders AS T
USING (SELECT * FROM raw.orders) AS S
ON T.order_id = S.order_id

WHEN MATCHED THEN UPDATE SET ...
WHEN NOT MATCHED THEN INSERT ...
WHEN NOT MATCHED BY SOURCE THEN DELETE;
```

That last clause makes the target **mirror** the source. Delete an order upstream
and it disappears downstream. Without it, deletions never propagate.

## What dbt generates

`default__get_merge_sql` produces exactly two `when` branches:

```sql
when matched then update set ...
when not matched then insert (...) values (...)
```

There is no `when not matched by source`, and nothing in the config surface adds
one. `merge_update_columns`, `merge_exclude_columns`, `incremental_predicates` —
none of them touch this.

The clause **does** appear in dbt's SQL, but only in the `insert_overwrite`
builder:

```sql
when not matched by source
     and <partition predicate>
    then delete
```

There it's scoped to a partition set, which is a different operation from
"anything missing from source".

## Your three options

**1. `insert_overwrite`, if the data is partitioned.**

If your source is a complete snapshot of some partition range, `insert_overwrite`
gives you mirror semantics *within* those partitions — everything in them is
replaced by what the model produced, deletions included.

```sql
{{ config(
    materialized='incremental',
    incremental_strategy='insert_overwrite',
    partition_by={'field': 'order_date', 'data_type': 'date'},
    partitions=['date_sub(current_date(), interval 1 day)', 'current_date()']
) }}
```

Use the static `partitions` form. With the dynamic form, a partition your model
produces *nothing* for is left untouched — which is precisely the deletion case
you're trying to handle. See
[B14](B14-when-the-range-can-empty.md).

This is the right answer when deletions are bounded in time. It's the wrong
answer if an order from three years ago can be deleted today, because you'd have
to overwrite three years of partitions to catch it.

**2. `materialized='table'`.**

If the source is a full snapshot and the table isn't enormous, rebuild it. Mirror
semantics for free, no drift, no silent failure modes.

People dismiss this too quickly. Run the numbers before ruling it out — a nightly
full rebuild of a few hundred GB is often cheaper than the engineering time spent
avoiding it, and it is *correct by construction*.

**3. Soft deletes.**

Change the shape of the problem: have upstream mark rows deleted rather than
removing them, then filter downstream.

```sql
select * from {{ ref('orders_raw') }}
where not is_deleted
```

Now deletion is an update, `merge` handles it natively, and you keep an audit
trail. Best long-term answer, and the one that requires a conversation with
whoever owns the source.

## What not to do

**Don't put the delete in a post-hook.** It's DML against the model's own table,
outside the materialization, running after the merge — and on failure you get a
partial state with no rollback. It also runs before `apply_grants` and hides a
core behaviour in a config string. See
[F17](../F-hooks/F17-when-a-hook-is-wrong.md).

**Don't ignore it and hope.** If the script had this clause, someone needed
deletions to propagate. Dropping it means your converted table accumulates rows
that no longer exist upstream — silently, and increasingly.

## Decide before you convert

This is an [A8](../A-assess/A8-estimate-risk.md) risk multiplier. Answer two
questions first:

1. **Can rows be deleted upstream?** If genuinely never, the clause is dead code
   and you can drop it — record that in [A9](../A-assess/A9-correctness-baseline.md).
2. **If they can, how far back?** Bounded ⇒ option 1. Unbounded ⇒ option 2 or 3.

Getting this wrong produces a table that's correct on day one and quietly wrong
by month six.

---

Previous: [B9 · `WHEN MATCHED THEN UPDATE` → update columns](B9-when-matched-update.md) ·
Next: [B11 · Conditional `WHEN MATCHED AND ...`](B11-conditional-when-matched.md)
