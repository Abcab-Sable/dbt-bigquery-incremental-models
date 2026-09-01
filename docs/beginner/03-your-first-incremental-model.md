# 3. Your first incremental model

Let's build one, run it twice, and look at exactly what happens each time.

## The starting point

Here's a plain, non-incremental model. Save it as `clean_clicks.sql`:

```sql
{{ config(materialized='table') }}

select
    click_id,
    user_id,
    clicked_at,
    page
from {{ source('raw', 'clicks') }}
where user_id is not null
```

Every `dbt run`, this reads all of `raw.clicks` and replaces `clean_clicks`
entirely. Correct, simple, and increasingly expensive — the problem from
[page 1](01-what-problem-are-we-solving.md).

## Making it incremental

Two changes. First the config:

```sql
{{ config(materialized='incremental') }}
```

Second — and this is the part people miss — **you have to tell dbt what "new"
means.** dbt will not guess:

```sql
{{ config(materialized='incremental') }}

select
    click_id,
    user_id,
    clicked_at,
    page
from {{ source('raw', 'clicks') }}
where user_id is not null

{% if is_incremental() %}
  and clicked_at > (select max(clicked_at) from {{ this }})
{% endif %}
```

Three new pieces of vocabulary in that block:

**`is_incremental()`** is true only when all of these hold: the table already
exists, it's a table (not a view), the model is materialized incremental, and
you're not doing a full refresh. So it's false on the very first run, and false
whenever you `--full-refresh`.

**`{{ this }}`** means "the table this model builds". Here, `clean_clicks`
itself. You're querying your own output to find where you got to.

**The `{% if %}` block** is templating, not SQL. dbt removes it before sending
anything to BigQuery. On a first run the extra `and ...` line simply isn't there.

## Run 1: the table doesn't exist

dbt checks for `clean_clicks`, doesn't find it, and takes the simplest path:
`CREATE TABLE AS` with your query. `is_incremental()` is false, so no filter is
added. Everything gets read and written.

```
raw.clicks:     5,000,000 rows
clean_clicks:   4,800,000 rows  (200,000 had a null user_id)
```

Identical to what the `table` materialization would have done. **An incremental
model's first run is always a full build.**

## Run 2: the table exists

Now `is_incremental()` is true, so the compiled SQL includes the filter:

```sql
select click_id, user_id, clicked_at, page
from raw.clicks
where user_id is not null
  and clicked_at > (select max(clicked_at) from `project`.`dataset`.`clean_clicks`)
```

Suppose 50,000 new clicks arrived. That query returns 50,000 rows instead of
5,050,000.

But dbt doesn't just insert them. It wraps them in a `MERGE`:

```sql
merge into clean_clicks as DBT_INTERNAL_DEST
    using ( <your select, from above> ) as DBT_INTERNAL_SOURCE
    on FALSE

when not matched then insert
    (click_id, user_id, clicked_at, page)
values
    (click_id, user_id, clicked_at, page)
```

Look closely at `on FALSE`.

## `on FALSE` — the most important two words on this page

The `on` clause is the matching rule. `FALSE` means **nothing ever matches**.
Every source row falls through to `when not matched`, and gets inserted.

Why would dbt do that? Because we never gave it a `unique_key`. Without one, dbt
has no way to know whether an incoming row is "the same" as an existing row — so
it doesn't try. It inserts everything.

There is no `when matched then update` clause in that statement at all. dbt omits
it entirely when there's no unique key.

**So this model is append-only**, and that's fine *here*, because our
`clicked_at > max(clicked_at)` filter guarantees we only ever fetch rows we've
never seen.

The danger is when those two things fall out of step. If your filter ever returns
a row you already have, you now have it twice, and nothing will tell you. Which
brings us to the next section.

## Adding a `unique_key`

Switch examples: orders, where rows genuinely change.

```sql
{{ config(
    materialized='incremental',
    unique_key='order_id'
) }}

select
    order_id,
    customer_id,
    status,
    updated_at
from {{ source('raw', 'orders') }}

{% if is_incremental() %}
  where updated_at > (select max(updated_at) from {{ this }})
{% endif %}
```

Now the generated `MERGE` looks different:

```sql
merge into orders as DBT_INTERNAL_DEST
    using ( <your select> ) as DBT_INTERNAL_SOURCE
    on (DBT_INTERNAL_SOURCE.order_id = DBT_INTERNAL_DEST.order_id)

when matched then update set
    order_id = DBT_INTERNAL_SOURCE.order_id,
    customer_id = DBT_INTERNAL_SOURCE.customer_id,
    status = DBT_INTERNAL_SOURCE.status,
    updated_at = DBT_INTERNAL_SOURCE.updated_at

when not matched then insert
    (order_id, customer_id, status, updated_at)
values
    (order_id, customer_id, status, updated_at)
```

The `on FALSE` became a real comparison, and the `when matched then update set`
block appeared. Now:

- Order 5 already exists, and arrives with `status = 'shipped'` → **updated**.
- Order 9 is brand new → **inserted**.

Walk it through with data. Before:

| order_id | status  | updated_at |
| -------- | ------- | ---------- |
| 5        | pending | 09:00      |
| 6        | shipped | 09:30      |

Incoming (`updated_at > 09:30`):

| order_id | status    | updated_at |
| -------- | --------- | ---------- |
| 5        | shipped   | 10:00      |
| 9        | pending   | 10:15      |

After:

| order_id | status  | updated_at |
| -------- | ------- | ---------- |
| 5        | shipped | 10:00      |
| 6        | shipped | 09:30      |
| 9        | pending | 10:15      |

Order 5 updated in place, order 6 untouched, order 9 added. That's the behaviour
people expect from "incremental", and you only get it with a `unique_key`.

## A wrinkle in the filter

Notice the filter is `updated_at > (select max(updated_at) from {{ this }})` —
strictly greater than. If two orders share the exact same `updated_at` as the
current maximum, and one of them arrives after the run, it will be skipped
forever.

The usual fix is a small overlap window:

```sql
{% if is_incremental() %}
  where updated_at >= (
      select timestamp_sub(max(updated_at), interval 1 hour) from {{ this }}
  )
{% endif %}
```

You reprocess an hour of data every run. With a `unique_key`, reprocessing is
harmless — those rows just update themselves to identical values. **Without** a
unique key, that same overlap produces an hour of duplicates every run.

This is the first place the pieces interact: your filter and your `unique_key` are
not independent choices.

## What to take away

- The first run is always a full build.
- `is_incremental()` is how you add "only new rows" filtering.
- **No `unique_key` means `on FALSE`, which means append-only.** Duplicates are
  your responsibility.
- **With a `unique_key`, existing rows update in place**, and overlapping windows
  become safe.
- Read the compiled SQL. Everything above is visible in `target/compiled/`.

So far, so good. Now the cost problem: this `MERGE` still has to search
`clean_clicks` for matches, and that means reading it. To make *that* cheap, you
need partitions.

---

Previous: [2. The words people will use at you](02-vocabulary.md) ·
Next: [4. Partitions, explained properly](04-partitioning-explained.md)
