# B16 · Deduplication scripts

> **Part B — Write-pattern archetypes** · Sourcing: `CRAFT`
> **The question:** my script dedups. Does the model still need to?

Maybe not. A dedup step is often a scar from the script's own non-idempotency,
and converting removes the cause. But sometimes the duplicates are real and
upstream, in which case it stays.

Work out which before deciding.

## The pattern

```sql
CREATE OR REPLACE TABLE analytics.orders AS
SELECT * EXCEPT(rn) FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY updated_at DESC) AS rn
    FROM analytics.orders_raw
)
WHERE rn = 1;
```

Or the modern form:

```sql
SELECT * FROM analytics.orders_raw
QUALIFY ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY updated_at DESC) = 1;
```

## Find out where the duplicates come from

Three sources, three different answers.

**The source genuinely has them.** A vendor double-sends; a stream delivers
at-least-once; a CDC feed emits one row per change. The duplicates are real input
and the dedup is real logic — **keep it**.

**The script created them.** An append with no key, plus an overlapping window or
a re-run. The duplicates are self-inflicted, and a `unique_key` removes the cause
entirely — **drop the dedup**.

**Nobody knows.** The most common. Test it:

```sql
select order_id, count(*) as n
from analytics.orders_raw
group by order_id
having count(*) > 1
order by n desc
limit 20;
```

Duplicates in the **raw source** ⇒ upstream, keep the dedup. Clean source but a
dedup step downstream ⇒ it was compensating for the script itself, and it can go.
That's [A6](../A-assess/A6-compensating-hacks.md).

## If they're upstream: dedup belongs early

Put it in a staging model, once, so everything downstream inherits clean data:

```sql
-- models/staging/stg_orders.sql
{{ config(materialized='view') }}

select * from {{ source('raw', 'orders') }}
qualify row_number() over (partition by order_id order by updated_at desc) = 1
```

Then downstream models just `ref('stg_orders')` and stop thinking about it. Dedup
repeated in five models is five places to get the tiebreak wrong.

## If they're self-inflicted: `unique_key` is the fix

```sql
{{ config(
    materialized='incremental',
    unique_key='order_id'
) }}
```

Now a re-read updates the existing row instead of adding a second one.

**Don't do both without thinking.** A `unique_key` plus a dedup in the model body
is fine — belt and braces — but if you're deduping *because* you have no key, add
the key and drop the dedup. Keeping both hides whether the key is working.

## The tiebreak needs care within a batch

`unique_key` handles duplicates **across runs**. It does not handle two rows with
the same key **in the same batch** — the `MERGE` sees both, and which one wins is
not something you should rely on.

So if a single run can contain two versions of one key, you need the `QUALIFY`
regardless of the `unique_key`:

```sql
select * from {{ source('raw', 'orders') }}
{% if is_incremental() %}
  where updated_at >= (select timestamp_sub(max(updated_at), interval 1 hour) from {{ this }})
{% endif %}
qualify row_number() over (partition by order_id order by updated_at desc) = 1
```

This combination — overlapping window, `unique_key`, in-batch dedup — is the
robust shape for an upsert from a messy source.

## Make the tiebreak deterministic

`ORDER BY updated_at DESC` alone is non-deterministic when two rows share a
timestamp. Different runs can pick different winners, which breaks
[idempotency](../E-translation/E7-idempotency-meaning.md) and any content parity
check.

Add a stable tiebreaker:

```sql
qualify row_number() over (
    partition by order_id
    order by updated_at desc, ingested_at desc, _file_name desc
) = 1
```

Anything guaranteed unique will do. Without it, `H3` comparisons will show
spurious differences and you'll spend a day chasing them.

## Test it afterwards

```sql
select order_id, count(*) as n
from {{ this }}
group by order_id
having count(*) > 1
```

Make it a `unique` test on the key so it stays true rather than being true once —
[H12](../H-verification/H12-tests-from-guarantees.md).

---

Previous: [B15 · `TRUNCATE` + `INSERT`](B15-truncate-insert.md) ·
**End of wave 2.** Back to [the backlog](../BACKLOG.md#delivery-waves)
