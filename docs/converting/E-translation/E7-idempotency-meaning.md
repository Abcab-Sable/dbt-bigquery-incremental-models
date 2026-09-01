# E7 · Idempotency: what it means for a converted model

> **Part E — Statement-level translation** · Sourcing: `CRAFT`
> **The question:** what should happen if this runs twice?

The same thing as running it once. That's the whole definition — and most scripts
don't satisfy it, which is why so many have a manual cleanup step nobody
documented.

## Why conversion forces the question

A script runs when someone runs it. A dbt model runs on a schedule, in CI, during
a backfill, and again when someone retries a failed run. It will be run twice,
and often three times in a day.

If running twice differs from running once, you don't have a bug that might
happen. You have one that will.

## Which strategies give it to you

| Setup | Idempotent? |
| --- | --- |
| `materialized='table'` | **Yes.** Full replace every time |
| `merge` with a `unique_key` | **Yes.** Second run updates rows to identical values |
| `merge` with **no** `unique_key` | **No.** `on FALSE` ⇒ every row inserts again |
| `insert_overwrite`, dynamic | **Yes** for the partitions produced — see below |
| `insert_overwrite`, static `partitions` | **Yes.** The listed partitions are replaced regardless |
| `append` shapes generally | **No**, unless your filter guarantees no overlap |

The `merge`-without-a-key row is the one that catches people, because it's the
default shape if you forget `unique_key`. See
[the balanced track](../../balanced/02-choosing-a-strategy.md#merge-without-a-unique_key-is-append-only).

## The caveat on dynamic `insert_overwrite`

It's idempotent for the partitions your model produces — run it twice, those
partitions get replaced twice with the same content.

It is **not** idempotent in the sense of making the table match your model,
because partitions your model didn't produce are never touched, no matter how
many times you run. That's [B14](../B-write-patterns/B14-when-the-range-can-empty.md),
and it's a different property than the one this page is about. Both matter.

## Where scripts lose it

**Append with an overlapping window.** The most common. A watermark of
`>= max - 1 hour` reprocesses an hour every run. With a `unique_key` that's free
insurance against late data. Without one it's an hour of duplicates, every run,
forever.

**A `DELETE` narrower than the `INSERT`.** The script deletes 3 days and inserts
7. Days 4–7 accumulate. Look for this whenever the two ranges aren't written
identically.

**Sequence or timestamp generation inside the model.**

```sql
select
    generate_uuid()      as row_id,       -- different every run
    current_timestamp()  as processed_at  -- different every run
from ...
```

The rows differ every run even when the input hasn't changed, which breaks both
idempotency and any content-based parity check in
[H3](../H-verification/H3-checksum-parity.md). If you need a stable id, hash
the business key instead:

```sql
to_hex(md5(concat(cast(order_id as string), '|', cast(line_no as string)))) as row_id
```

**Reading `current_date()` in a way that shifts the window.** A model whose
output depends on when it ran isn't reproducible. Sometimes unavoidable; make it
deliberate rather than accidental.

## The distinction worth holding

There are two different properties, and people conflate them:

**Run-twice idempotency** — running now and again in five minutes gives the same
table. This is what you must have.

**Reproducibility** — running today and again next month over the same logical
period gives the same output. Stronger, and not always achievable when upstream
data mutates. Worth knowing which one you have.

A model can be perfectly idempotent and still not reproducible, because upstream
changed. That's normal. A model that isn't idempotent is broken.

## Design for it

- Give every `merge` model a `unique_key`, even when you think inserts are
  unique. It costs nothing and removes a whole failure class.
- Make delete and insert ranges identical, expressed once.
- Keep non-deterministic functions out of model bodies.
- Prefer `insert_overwrite` with explicit `partitions` when the period boundary
  is what you actually mean.

Then prove it rather than assuming — [E8](E8-idempotency-proving.md).

---

Previous: [E6 · Hardcoded dates and backfill parameters](E6-hardcoded-dates.md) ·
Next: [E8 · Idempotency: proving it](E8-idempotency-proving.md)
