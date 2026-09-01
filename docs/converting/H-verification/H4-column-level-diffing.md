# H4 · Column-level diffing

> **Part H — Proving correctness** · Sourcing: `CRAFT`
> **The question:** which column is wrong, and by how much?

You know the tables differ ([H3](H3-checksum-parity.md)) and which partitions.
This finds the column, which is what tells you what to fix.

## Join on the key, compare each column

```sql
with old as (select * from analytics.orders),
     new as (select * from analytics_dbt.orders)
select
    countif(old.status     is distinct from new.status)      as status_diffs,
    countif(old.amount     is distinct from new.amount)      as amount_diffs,
    countif(old.customer_id is distinct from new.customer_id) as customer_diffs,
    countif(old.updated_at is distinct from new.updated_at)  as updated_at_diffs,
    count(*) as compared
from old
full outer join new using (order_id);
```

`IS DISTINCT FROM` treats two nulls as equal, which is what you want here — a
plain `!=` returns `NULL` when either side is null and silently undercounts.
That's [H8](H8-reconciling-nulls.md).

The output is a per-column count, so you immediately know whether it's one column
or all of them. One column ⇒ a logic difference. All columns ⇒ the join key is
wrong, or the row sets differ.

## Generate it rather than typing it

For a wide table, write the comparison with a macro:

```sql
{% macro compare_columns(old_rel, new_rel, key, exclude=[]) %}
  {%- set cols = adapter.get_columns_in_relation(ref(new_rel)) -%}
  select
  {%- for c in cols if c.name not in exclude and c.name != key %}
    countif(o.{{ c.name }} is distinct from n.{{ c.name }}) as {{ c.name }}_diffs
    {{- ',' if not loop.last }}
  {%- endfor %}
  from {{ old_rel }} o full outer join {{ new_rel }} n using ({{ key }})
{% endmacro %}
```

Run it with `dbt run-operation`, or paste the compiled output. Manually listing
forty columns is where mistakes get made — usually by omitting the column that
turns out to be wrong.

## Look at examples, not just counts

A count tells you a column differs; the rows tell you why:

```sql
select
    order_id,
    old.amount as old_amount,
    new.amount as new_amount,
    new.amount - old.amount as delta
from analytics.orders old
full outer join analytics_dbt.orders new using (order_id)
where old.amount is distinct from new.amount
order by abs(new.amount - old.amount) desc
limit 20;
```

Sorting by the largest difference is the fastest route to a diagnosis. The
pattern usually names the cause:

| Pattern | Likely cause |
| --- | --- |
| Always exactly zero vs null | Null handling — [H8](H8-reconciling-nulls.md) |
| Differences in the last decimal place | Precision — [H9](H9-reconciling-numeric-precision.md) |
| Off by exactly one hour, or a whole day | Timezone — [H10](H10-reconciling-timestamps.md) |
| New is a subset of old | A filter you added, or a predicate window |
| Only recent rows differ | A boundary, not a bug |
| Only one column, consistently | A `merge_update_columns` change — [B9](../B-write-patterns/B9-when-matched-update.md) |
| Only rows that were updated | Same |

## The `created_at` case

Worth checking explicitly, because row counts and set comparisons can both pass
while it's wrong.

If your script's `MERGE` had a narrow `UPDATE SET` and you didn't carry it across,
dbt updates **every** column — including ones the script deliberately preserved.
The classic casualty is a first-seen timestamp:

```sql
select count(*) as created_at_moved
from analytics.orders old
join analytics_dbt.orders new using (order_id)
where old.created_at is distinct from new.created_at;
```

Non-zero on a column that should never change ⇒
[B9](../B-write-patterns/B9-when-matched-update.md).

## Scope it to what matters

Diffing every column of a billion-row table is expensive. Bound it:

```sql
where old.order_date >= '2026-08-01'
```

Compare a representative window, not everything. If that window is clean and the
per-partition fingerprints from [H3](H3-checksum-parity.md) match elsewhere,
that's strong evidence — and cheap.

---

Previous: [H3 · Checksum and hash parity](H3-checksum-parity.md) ·
Next: [H5 · Shadow mode](H5-shadow-mode.md)
