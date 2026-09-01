# A3 · Classify by write pattern

> **Part A — Assess before you convert** · Sourcing: `CRAFT`
> **The question:** what kind of script is this, and which conversion applies?

This is the routing step. Classify first and the rest of the track tells you
which page to read. Skip it and you'll pick a strategy by vibe, which is how
people end up with `merge` on a two-billion-row table.

## The six classes

Ignore what the script is *about*. Look only at **how it writes**.

| Class | Signature | Converts to |
| --- | --- | --- |
| **Replace** | `CREATE OR REPLACE TABLE`, or `TRUNCATE` then `INSERT` | `materialized='table'` |
| **Append** | `INSERT INTO ... SELECT`, no delete | incremental, append semantics |
| **Upsert** | `MERGE`, or `UPDATE` then `INSERT` | incremental + `unique_key` |
| **Delete-insert** | `DELETE WHERE <range>` then `INSERT` | `insert_overwrite` |
| **DDL-only** | `ALTER`, `CREATE VIEW`, options, grants | configs, or hooks |
| **Side-effect** | `EXPORT DATA`, calls out, writes metadata | hooks, or stays outside dbt |

## Finding the class

Read the statements that **write**, and ignore everything else. In practice:

```bash
grep -inE '^\s*(create|insert|merge|update|delete|truncate|alter|export|load)' script.sql
```

That output is your classification, in order. A clean single-class script is
obvious from it. Most real scripts aren't clean.

## Reading the combinations

Real scripts mix classes. The combination usually tells you more than any single
statement does.

**`DELETE` + `INSERT` on the same table** — delete-insert, and the single most
important conversion in this track. Look hard at the `DELETE`'s `WHERE`:

- Bounded by a date range ⇒ [B13](../B-write-patterns/B13-delete-insert-to-insert-overwrite.md), `insert_overwrite`.
- Bounded by keys matching the insert ⇒ that's an upsert written longhand.
  Convert as [B8](../B-write-patterns/B8-merge-on-clause-to-unique-key.md), not
  as delete-insert.
- Unbounded (`DELETE FROM t` with no `WHERE`) ⇒ that's a replace. Use `table`.

**`CREATE OR REPLACE` on a table nothing else writes to** — replace. Genuinely
just `materialized='table'`. Don't make it incremental because it's big; see
[A7](A7-what-not-to-convert.md).

**`INSERT` with a `WHERE` referencing the target's own max** — append with a
watermark. The most common incremental shape there is.

**`MERGE` plus a separate `DELETE`** — upsert *and* a retention policy, tangled.
Two concerns, and the retention part usually isn't a model at all.

**Several `INSERT`s into different tables** — this is not one script. It's a
pipeline that was never separated, and it becomes several models.

## The questions that decide the strategy

Once classified, three questions settle the rest.

**1. Does the script ever modify a row it wrote previously?**
Yes ⇒ you need `unique_key` and `merge`. No ⇒ `insert_overwrite` is on the table
and will usually be cheaper.

**2. Is the write bounded by a date or partition range?**
Yes ⇒ `insert_overwrite`, and you already know the partition column. No ⇒
`merge`, or reconsider whether it should be a `table`.

**3. Can a period legitimately produce zero rows?**
This is the question nobody asks, and it decides whether the naive conversion is
*correct*. If yes, dynamic `insert_overwrite` will silently leave stale data
behind — that's [B14](../BACKLOG.md#part-b--write-pattern-archetypes) and it is
the reason this track exists.

## Record the answer

Write the classification down next to the script, with the evidence:

```
orders_daily.sql
  class:       delete-insert
  evidence:    DELETE WHERE order_date >= @start; INSERT SELECT ... 
  target:      analytics.orders_daily, partitioned on order_date (day)
  modifies existing rows?  no — full rebuild of the range
  can a day be empty?      YES — refunds-only days produce nothing
  → insert_overwrite, static partitions (see B14)
```

That last line is the whole point of Part A. You now know not just the strategy
but the one thing that would have made the obvious version wrong.

Carry this into [A7](A7-what-not-to-convert.md) to decide whether it's worth
converting at all, then into the matching Part B page.

## When it won't classify

Some scripts resist. Usually one of:

- **It's doing three things.** Split it on paper first; classify the pieces.
- **The writes are dynamic** (`EXECUTE IMMEDIATE`). Classify what it *generates*,
  not the generator.
- **It writes nothing.** It's a check, an export, or an alert. That's the
  side-effect class, and it probably isn't a model — [A7](A7-what-not-to-convert.md).

---

Previous: [E1 · One statement per model](../E-translation/E1-one-statement-per-model.md) ·
Next: [A7 · Decide what not to convert](A7-what-not-to-convert.md)
