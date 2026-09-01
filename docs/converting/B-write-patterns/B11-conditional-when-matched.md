# B11 · Conditional `WHEN MATCHED AND ...` clauses

> **Part B — Write-pattern archetypes** · Sourcing: `SRC`
> **The question:** my `MERGE` only updates when a condition holds. Where does that go?

Into the model body, not the config. dbt generates one unconditional `when
matched` branch, so a conditional update has to become a decision about which
rows you emit.

## The pattern

```sql
WHEN MATCHED AND S.updated_at > T.updated_at THEN UPDATE SET ...
```

Only overwrite if the incoming row is newer. Guards against out-of-order arrivals
overwriting good data with stale data.

Or the two-branch form:

```sql
WHEN MATCHED AND S.is_deleted THEN DELETE
WHEN MATCHED THEN UPDATE SET ...
```

## What dbt gives you

One branch, no condition:

```sql
when matched then update set <columns>
```

`default__get_merge_sql` has no mechanism for a condition on the `when matched`
clause. `incremental_predicates` appends to the **`ON`** clause, which is not the
same thing — it decides what *matches*, not what happens once matched.

## Moving the condition into the model

The usual translation for the newer-wins case is deduplication in the model, so
only the winning row is ever presented to the merge:

```sql
{{ config(
    materialized='incremental',
    unique_key='order_id'
) }}

with incoming as (
    select * from {{ source('raw', 'orders') }}
    {% if is_incremental() %}
      where updated_at >= (
          select timestamp_sub(max(updated_at), interval 1 hour) from {{ this }}
      )
    {% endif %}
)
select * from incoming
qualify row_number() over (partition by order_id order by updated_at desc) = 1
```

This handles duplicates *within the batch*. It does not, by itself, stop a batch
row older than the stored row from overwriting it.

For that, compare against the target explicitly:

```sql
select i.*
from incoming i
{% if is_incremental() %}
left join {{ this }} t using (order_id)
where t.order_id is null
   or i.updated_at > t.updated_at
{% endif %}
```

Now only genuinely newer rows reach the merge, and the unconditional `when
matched` is safe. Note this reads the target, so bound it with a partition filter
if the table is large.

## The delete branch

`WHEN MATCHED AND S.is_deleted THEN DELETE` has no dbt equivalent —
same limitation as [B10](B10-not-matched-by-source.md). Options:

- **Filter deleted rows out** and accept they linger in the target (soft-delete
  semantics, if downstream also filters)
- **`insert_overwrite`** over the affected partitions, so absence means deletion
- **Keep the flag** and have every consumer filter on it — often the cleanest

## Is the condition still load-bearing?

Worth asking, because these clauses are frequently
[compensating hacks](../A-assess/A6-compensating-hacks.md). `S.updated_at >
T.updated_at` usually got added after an incident where out-of-order data
corrupted a table.

If the source is now ordered, or the pipeline that caused it is gone, the
condition may be dead. Don't remove it during the conversion — port it, note it,
and test removing it afterwards as a separate change
([K11](../BACKLOG.md#part-k--anti-patterns)).

## Verify the behaviour, not the row count

Row counts won't move whichever way this goes. Test it directly:

1. Insert a row with a known `updated_at`.
2. Present an *older* version of the same row.
3. Check the target still holds the newer values.

If it doesn't, the condition didn't survive the conversion.

---

Previous: [B10 · `WHEN NOT MATCHED BY SOURCE THEN DELETE`](B10-not-matched-by-source.md) ·
Next: [B12 · Extra `ON` predicates → `incremental_predicates`](B12-extra-predicates.md)
