# I1 · Conversion order: leaves first or roots first

> **Part I — Migration strategy** · Sourcing: `CRAFT`
> **The question:** forty scripts. Which one do I convert first?

Roots first — the things nothing else depends on the *output* of, sitting closest
to your raw data. Then work downstream.

## Why roots first

A converted model reads its inputs via `ref()` or `source()`. If you convert a
mart first, its inputs are still script-produced tables, so they're all
`source()`s — and every one of them becomes a `ref()` later, in a second change.

Convert roots first and each new model's inputs are already `ref()`-able. You
touch each reference once.

```
raw.events  →  [stg_events]  →  [int_enriched]  →  [daily_summary]
               convert 1st       convert 2nd        convert 3rd
```

## The exception: risk first

That's the mechanical answer. The practical one balances it against
[A8](../A-assess/A8-estimate-risk.md).

Start with something **low-risk and low-difficulty**, wherever it sits in the
graph. You need to establish the pattern — the naming convention, the review
process, the parity workflow, who signs off — on something nobody is watching.

Two or three of those, then switch to roots-first and work systematically.

## Deferred conversions are fine

You don't have to convert a whole chain at once. A script you haven't converted
yet is a `source()`:

```yaml
sources:
  - name: legacy
    schema: analytics
    tables:
      - name: daily_events
        description: >
          Produced by daily_events.sql (scheduled query, 03:00 UTC).
          Converting in DATA-2288 — becomes ref() then.
```

Downstream models get lineage and freshness immediately. When you convert the
script, delete the source and swap every `source('legacy', ...)` for `ref(...)`
in one change — [E3](../E-translation/E3-ref-vs-source.md).

Nothing should reference both:

```bash
grep -rn "source('legacy', 'daily_events')" models/
```

## Sequence from the dependency map

[A2](../A-assess/A2-map-dependencies.md) gives you the real graph. Convert in
topological order, but pull two categories forward:

**Shared inputs.** A table five scripts read is worth converting early — five
future conversions get simpler.

**Things with undeclared dependencies.** Converting these makes an invisible edge
visible, which is a correctness improvement independent of everything else.

And push one category back: **circular dependencies**. dbt raises
`CompilationError: Found a cycle` and won't accept them, so those need designing
out before they can be converted at all —
[E2](../E-translation/E2-ordering-by-ref.md).

## Batch size

Convert in small batches — three to five scripts, then verify and cut over. Not
forty at once.

A large batch means a large parity exercise, many simultaneous differences to
explain, and no way to attribute a problem to a particular change. Small batches
make [H11](../H-verification/H11-differences-that-should-exist.md) tractable.

## Keep a visible list

The inventory from [A1](../A-assess/A1-inventory.md), annotated:

```
daily_events.sql       converted, shadow running    DATA-2288
load_users.sql         converted, cut over 09-02    DATA-2289
build_summary.sql      in progress
legacy_recon.sql       NOT CONVERTING (A7 — one-off, unowned)
vendor_sync.sql        blocked — circular dep with build_summary
```

Migrations stall when nobody can see the state. This list is also what tells you
when you're done — and when to stop, because some entries should stay
`NOT CONVERTING`.

---

Next: [I2 · The strangler pattern](I2-strangler-pattern.md) ·
Back to [the backlog](../BACKLOG.md)
