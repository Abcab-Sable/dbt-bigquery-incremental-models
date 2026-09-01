# 4. Partitions, explained properly

Two of the three strategies **require** partitioning, and the third one is
usually too expensive without it. So this page is not optional background.

## The filing cabinet

Imagine a table as a filing cabinet full of paper.

An **unpartitioned** table is one enormous drawer. To find all of Tuesday's
clicks, you take out every sheet and check its date. Even if Tuesday is 1% of the
paper, you handled 100% of it.

A **partitioned** table has one drawer per day. Tuesday's clicks are in the
drawer labelled Tuesday. You open one drawer, take out its contents, and you're
done. You physically never touched the other drawers.

That's it. That's partitioning. BigQuery stores rows in separate chunks based on
a column's value, and can read individual chunks without touching the rest.

## Why you should care: the bill

BigQuery charges by **bytes read**. Not rows returned, not query duration —
bytes scanned.

A 2 TB table of two years of clicks, and this query:

```sql
select count(*) from clicks where event_date = '2026-08-31'
```

- **Unpartitioned**: BigQuery reads all 2 TB to check each row's date. You pay
  for 2 TB.
- **Partitioned on `event_date`**: BigQuery opens one day's chunk, ~2.8 GB. You
  pay for 2.8 GB.

Same query, same answer, roughly **700× difference** in cost.

That skipping is called **pruning**, or partition elimination. It is the single
most important performance concept in BigQuery, and it's why incremental models
on BigQuery are built around partitions rather than around rows.

## Turning it on

```sql
{{ config(
    materialized='incremental',
    partition_by={
        'field': 'event_date',
        'data_type': 'date',
        'granularity': 'day'
    }
) }}
```

Three settings:

**`field`** — the column to partition on. Must exist in your model's output.

**`data_type`** — what kind of column it is. `date`, `timestamp`, `datetime`, or
`int64`. Defaults to `date`.

**`granularity`** — how big each chunk is: `hour`, `day`, `month`, or `year`.
Defaults to `day`.

## Choosing a granularity

The instinct is "smaller is better, more precision". That instinct is wrong, and
it's the most common partitioning mistake.

BigQuery has a hard limit of **4,000 partitions per table**. Do the arithmetic
before you choose:

| Granularity | Partitions per year | Hits 4,000 after |
| --- | --- | --- |
| `hour` | 8,760 | **5.5 months** |
| `day` | 365 | 10.9 years |
| `month` | 12 | 333 years |
| `year` | 1 | a long time |

Hourly partitioning breaks in under six months. Unless you have a genuine need
and a retention policy that drops old partitions, **use `day`**. It's the right
answer nearly always.

The other reason to avoid tiny partitions: each one carries overhead. Thousands
of small partitions query *slower* than a sensible number of larger ones.

## What "your query prunes" actually requires

Partitioning the table is only half the job. BigQuery must also be able to *work
out* which partitions your query needs, and it has to do that **before** running
the query.

This prunes — a plain comparison on the partition column:

```sql
where event_date = '2026-08-31'
where event_date between '2026-08-01' and '2026-08-31'
where event_date >= current_date() - 7
```

This does **not** prune — the column is wrapped in a function, so BigQuery can't
reason about it:

```sql
where cast(event_date as string) = '2026-08-31'
where format_date('%Y-%m-%d', event_date) = '2026-08-31'
```

Both return the right answer. Both read the entire table. There's no error, no
warning — just a bill.

The rule of thumb: **keep the partition column bare on one side of the
comparison.** Transform the other side as much as you like.

This is worth internalising because dbt generates predicates on your partition
column automatically, and whether those prune is exactly why the strategies have
the shape they do.

## `require_partition_filter`, and a trap

You can force people to filter:

```sql
{{ config(
    partition_by={'field': 'event_date', 'data_type': 'date'},
    require_partition_filter=true
) }}
```

Now any query without a partition filter is rejected. A good guardrail against
someone accidentally scanning two years.

**But it does not make your own incremental model cheap.** When dbt builds a
`MERGE` against a table with this setting, it adds a predicate that satisfies the
rule without restricting anything:

```sql
(DBT_INTERNAL_DEST.event_date is null or DBT_INTERNAL_DEST.event_date is not null)
```

Every row is either null or not null, so that's always true for everything. It
ticks BigQuery's box and prunes nothing.

If you set `require_partition_filter` expecting cheaper merges, you'll be
disappointed. It protects you from *other people's* queries. Your own model still
needs a real bound.

## Clustering, briefly

Clustering sorts data *within* each partition:

```sql
{{ config(
    partition_by={'field': 'event_date', 'data_type': 'date'},
    cluster_by=['user_id']
) }}
```

Now within each day, rows are ordered by `user_id`, so filtering on `user_id`
reads less of the partition.

Useful, but secondary. Partitioning changes how incremental strategies *behave*;
clustering only changes how fast they run. One thing to know: **changing either
one later has consequences**, covered on [the next page](05-the-three-strategies.md)
and in [when things go wrong](06-when-things-go-wrong.md#the-full-refresh-that-dropped-the-table).

## Integers, briefly

You can partition on a number instead of a date:

```sql
partition_by={
    'field': 'customer_id',
    'data_type': 'int64',
    'range': {'start': 0, 'end': 1000000, 'interval': 1000}
}
```

That's 1,000 partitions of 1,000 customer IDs each. Same 4,000-partition limit
applies, so `end / interval` must stay under it.

Rarer than date partitioning, and it has a version-specific performance quirk
noted in the [balanced track](../balanced/06-partition-config.md#int64-range-partitions-get-boundary-normalisation).
If you're partitioning on dates — and you probably are — you can ignore it.

## The summary

- Partitioning splits a table into chunks so BigQuery can skip most of them.
- Skipping is called pruning, and it's where the cost saving comes from.
- **Use `day` granularity** unless you have a specific reason not to. 4,000
  partition limit.
- Pruning needs the partition column **bare** in the comparison.
- `require_partition_filter` guards against others, not against your own model.

Now you have everything needed for the strategies.

---

Previous: [3. Your first incremental model](03-your-first-incremental-model.md) ·
Next: [5. The three strategies](05-the-three-strategies.md)
