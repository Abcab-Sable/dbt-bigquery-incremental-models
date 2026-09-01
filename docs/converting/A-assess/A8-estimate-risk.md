# A8 · Estimate conversion risk

> **Part A — Assess before you convert** · Sourcing: `CRAFT`
> **The question:** which of these should I do first, and which will hurt?

You have an inventory ([A1](A1-inventory.md)), dependencies
([A2](A2-map-dependencies.md)), classifications ([A3](A3-classify-by-write-pattern.md)),
and known unknowns ([A5](A5-hidden-state.md), [A6](A6-compensating-hacks.md)).
This turns that into an order.

## Score on two axes

Risk is not the same as effort, and conflating them is why migrations stall on
the wrong thing.

**Blast radius** — what breaks if this is wrong.

| | |
| --- | --- |
| Low | Nothing reads it but one dashboard nobody opens |
| Medium | Feeds other tables, or a team's daily reporting |
| High | Feeds finance, external reporting, or a customer-facing surface |

**Conversion difficulty** — how likely the mapping is to change behaviour.

| | |
| --- | --- |
| Low | Single class, `CREATE OR REPLACE` or a clean `MERGE`, one owner, no hidden state |
| Medium | Delete-insert, or multi-statement, or a known compensating hack |
| High | Procedural logic, second writer, no owner, or an empty-period case |

## The four quadrants

**Low risk, low difficulty** ⇒ **start here.** Convert two or three to establish
the pattern, the review process, and the parity workflow before anything
important is at stake.

**Low risk, high difficulty** ⇒ good practice ground. If a conversion is going to
teach you something painful, learn it on something nobody's watching.

**High risk, low difficulty** ⇒ the best value in the migration. Do these once
the process is proven — usually second wave.

**High risk, high difficulty** ⇒ last, and deliberately. These need shadow
running ([H5](../H-verification/H5-shadow-mode.md)), a rollback plan
([I6](../I-migration/I6-rollback-keeping-script.md)), and a named owner. Some should
be redesigned rather than converted.

## The specific risk multipliers

Score these separately, because each one has bitten a real migration:

| Signal | Why it raises risk |
| --- | --- |
| **A period can legitimately be empty** | [B14](../B-write-patterns/B14-when-the-range-can-empty.md). Silent wrong output. The highest-value flag in this list |
| **No owner** | You cannot define correct, so you cannot verify |
| **Second writer on the target** | dbt will overwrite it |
| **Nullable columns in the intended `unique_key`** | Duplicates on every run |
| **Procedural control flow** | May not be convertible at all |
| **Target isn't partitioned but needs to be** | A table migration, not just a conversion |
| **Feeds something you don't control** | Cutover needs coordination — [I5](../I-migration/I5-notifying-consumers.md) |

Any two of these together and it belongs in the high-difficulty quadrant
regardless of how simple the SQL looks.

## Keep it rough

This is a sorting exercise, not an estimate. Two axes, three levels, plus the
multipliers above. Anything more elaborate takes longer than converting the first
few scripts would have.

The output you want is a list in the order you'll work through it, with the
scary ones explicitly deferred and the reason recorded. That order feeds
[I1](../I-migration/I1-conversion-order.md).

---

Previous: [A6 · Find the compensating hacks](A6-compensating-hacks.md) ·
Next: [E2 · Ordering by `ref()`](../E-translation/E2-ordering-by-ref.md)
