# K11 · Converting and optimising in the same change

> **Part K — Anti-patterns** · Sourcing: `CRAFT`
> **The question:** I'm rewriting it anyway. Why not improve it while I'm here?

Because then a difference could be the conversion or the improvement, and you
can't tell which. Every diff becomes ambiguous exactly when you most need clarity.

## The shape

While converting, you also:

- tidy the SQL and rename columns
- fix a bug you spotted
- change `FLOAT64` to `NUMERIC`
- add a filter that "should have been there"
- restructure the joins
- repartition the table

Then parity fails, and you have six candidate causes and no way to bisect.

## Why it's expensive

**Attribution.** [H11](../H-verification/H11-differences-that-should-exist.md)
depends on predicting differences. If you changed six things, every difference has
six possible sources.

**Rollback.** [I6](../I-migration/I6-rollback-keeping-script.md) assumes going
back to the script restores prior behaviour. If the model also fixed a bug,
rolling back reintroduces it — and consumers may have adapted.

**Review.** A reviewer can check "does this model match the script". They cannot
easily check "does this model match the script except for six intentional
changes".

**Sign-off.** [H13](../H-verification/H13-sign-off.md) asks someone to accept the
differences. A long list of entangled changes is much harder to accept than "it
matches".

## The sequence

1. **Convert faithfully.** Port compensating hacks
   ([K6](K6-porting-the-bug.md)), keep column names, keep the order of
   operations. Aim for equivalence.
2. **Verify and cut over.** Parity, shadow, sign-off, retire the script.
3. **Then improve**, one change at a time, each verified against the now-stable
   model.

Slower in calendar time, much faster in debugging time — and each step is
independently reversible.

## Specific things to defer

| Tempting | Why defer |
| --- | --- |
| Renaming columns | Breaks consumers, and drops [policy tags](../D-data-movement/D11-policy-tags-rls.md) |
| Fixing a bug | Produces a diff you must explain — [K6](K6-porting-the-bug.md) |
| Changing types | Downstream casts and joins may depend on them |
| Repartitioning | Forces drop-and-recreate — [D6](../D-data-movement/D6-partitioning-ddl.md) |
| Restructuring joins | Silently changes null handling — [H8](../H-verification/H8-reconciling-nulls.md) |
| Reordering operations | Changes float results — [H9](../H-verification/H9-reconciling-numeric-precision.md) |
| Consolidating schedules | Timing changes look like data changes — [G2](../G-scheduling/G2-consolidating-schedules.md) |
| `table` → `incremental` | Two risk profiles — [K3](K3-unnecessary-incremental.md) |

That last one is worth stating plainly: **convert to `table` first, make it
incremental second.** The first change proves the SQL; the second is then a pure
optimisation you can verify against the table version.

## The legitimate exceptions

**A bug severe enough that reproducing it is unacceptable.** Fix it, document it
loudly as an intended difference, get it agreed before cutover
([I5](../I-migration/I5-notifying-consumers.md)).

**Changes with no observable effect** — formatting, comments, CTE names. Free,
though they do make the diff against the script noisier.

**Things the conversion forces.** Adding a partition column for
`insert_overwrite` isn't optional ([B13](../B-write-patterns/B13-delete-insert-to-insert-overwrite.md)).

## Keep the improvement list

You'll spot plenty worth doing. Write it down rather than doing it:

```
daily_events — after conversion:
  - amount is FLOAT64, should be NUMERIC (DATA-2341)
  - vendor 892 exclusion: nobody knows why, test removing (DATA-2342)
  - hourly partitioning unnecessary, move to daily (DATA-2343)
  - three near-identical CTEs could be one (DATA-2344)
```

Four tickets, each independently verifiable. That list is a genuine deliverable of
the conversion — it's knowledge that only surfaces while reading the code this
closely.

---

Previous: [K10 · No tests, because the script had none](K10-no-tests.md) ·
Next: [K12 · Trusting a green run](K12-trusting-green-runs.md)
