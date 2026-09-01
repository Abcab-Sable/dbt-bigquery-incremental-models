# B9 · `WHEN MATCHED THEN UPDATE` → `merge_update_columns`

> **Part B — Write-pattern archetypes** · Sourcing: `SRC`
> **The question:** my `MERGE` only updates three columns. dbt updates all of them. Does that matter?

Often yes. This is the clause where a faithful-looking conversion quietly changes
what happens to your data.

## The default is "everything"

Your script:

```sql
WHEN MATCHED THEN UPDATE SET
    status     = S.status,
    updated_at = S.updated_at
```

dbt, with no extra config:

```sql
when matched then update set
    order_id = DBT_INTERNAL_SOURCE.order_id,
    customer_id = DBT_INTERNAL_SOURCE.customer_id,
    status = DBT_INTERNAL_SOURCE.status,
    amount = DBT_INTERNAL_SOURCE.amount,
    created_at = DBT_INTERNAL_SOURCE.created_at,
    updated_at = DBT_INTERNAL_SOURCE.updated_at
```

From `get_merge_update_columns`: with neither config set, `update_columns`
defaults to every destination column.

**If your script deliberately left a column alone, dbt will now overwrite it.**

## Why a script narrows the list

Usually one of:

- **A first-seen timestamp.** `created_at` must not change on update. Overwrite
  it and you've lost the acquisition date.
- **A column enriched elsewhere.** Another job populates it; this one must not
  clobber it. That's a second writer — [A5](../A-assess/A5-hidden-state.md).
- **A manually corrected value.** Someone fixed a row by hand and the script was
  taught not to undo it.

All three are silent when broken. The value is simply replaced, and nothing
flags it.

## The two configs

**Allow-list** — update only these:

```sql
{{ config(
    materialized='incremental',
    unique_key='order_id',
    merge_update_columns=['status', 'updated_at']
) }}
```

**Deny-list** — update everything except these:

```sql
merge_exclude_columns=['created_at']
```

Setting **both raises a compiler error**: *Model cannot specify
merge_update_columns and merge_exclude_columns.*

## Prefer the deny-list

The two are not symmetric in the source. `merge_exclude_columns` rebuilds the
list from `dest_columns` and takes `column.quoted`, so identifiers come out
properly quoted. `merge_update_columns` is used **as you wrote it** — your raw
strings go straight into the `SET` clause.

Practical consequences:

- A column name needing quoting works under exclude, may not under update
- Add a column to the model and the **allow-list silently won't update it** —
  you have to remember to edit the config
- The deny-list picks up new columns automatically, which is usually what you want

Use `merge_exclude_columns` unless you specifically want new columns to default
to not-updated.

## Map it directly

Compare your `SET` clause against the model's full column list:

```sql
select column_name
from `project.analytics.INFORMATION_SCHEMA.COLUMNS`
where table_name = 'orders'
order by ordinal_position;
```

Anything in that list but absent from your `SET` goes in
`merge_exclude_columns`. Do this by reading, not by memory — a missed column is
invisible until someone notices a `created_at` that keeps moving.

## Verify

```bash
dbt compile --select orders
```

Read the generated `update set` block and check it matches your script's, column
for column. Then test the specific behaviour:

1. Note a row's `created_at`.
2. Cause an update to that row.
3. Check `created_at` is unchanged.

A count-based parity check ([H2](../H-verification/H2-row-count-parity.md)) will
**not** catch this — the row count is identical either way. It needs a column-level
comparison, which is [H4](../BACKLOG.md#part-h--proving-correctness).

---

Previous: [B7 · Watermark in a separate state table](B7-external-watermark.md) ·
Next: [B10 · `WHEN NOT MATCHED BY SOURCE THEN DELETE`](B10-not-matched-by-source.md)
