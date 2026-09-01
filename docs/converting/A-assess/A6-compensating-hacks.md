# A6 · Find the compensating hacks

> **Part A — Assess before you convert** · Sourcing: `CRAFT`
> **The question:** which parts of this script are fixing someone else's bug?

Scripts grow patches. A `DELETE` that cleans up yesterday's double-insert. A
`WHERE` excluding a vendor whose data was malformed in 2023. A `DISTINCT` nobody
can justify. Each was a rational response to a real problem, and none of it is
documented.

Port them literally and you carry the bug forward with a clean face on it. That's
[K6](../K-antipatterns/K6-porting-the-bug.md), and conversion is the one moment
you're guaranteed to be reading the code closely enough to catch it.

## The signatures

**A `DISTINCT` or `QUALIFY ROW_NUMBER()` on data that should already be unique.**
Almost always compensating for duplicates from upstream — or from the script's
own non-idempotency. If the script appends without a key and someone re-ran it
once, the dedup is the scar.

Ask: *if I removed this, would there be duplicates?* If yes, find out where they
come from. That's the actual bug, and [B16](../B-write-patterns/B16-deduplication.md)
covers where dedup belongs afterwards.

**A `DELETE` that isn't part of the write pattern.** A delete-insert has a delete
that matches its insert range. A delete that removes something *else* — a
specific vendor, a date range, rows matching a condition — is a patch.

**Hardcoded exclusions.**

```sql
where vendor_id not in (447, 892)      -- ???
  and event_date != '2024-11-03'
```

Every one of these encodes an incident. Find out whether the condition still
applies. Usually nobody knows, which is itself the finding.

**Defensive `COALESCE` on columns that shouldn't be null.** Someone got null-
poisoned once. Either the upstream is genuinely nullable — in which case model it
properly — or it isn't and this is masking a real problem.

**A retry, a sleep, or a second identical statement.** Compensating for a race
the author couldn't fix. Under dbt the ordering is explicit, so the race may
simply not exist any more — but verify rather than assume.

**Widened date ranges.** `interval 7 day` where the schedule is daily. Usually
compensating for late-arriving data, which is legitimate — but it should be a
documented decision, not folklore. See [G8](../G-scheduling/G8-late-arriving-data.md).

## The triage

For each one, three outcomes:

**Still needed, and understood** ⇒ port it, with a comment explaining why. The
comment is the deliverable; it's what stops the next person deleting it.

```sql
-- Vendor 447 double-sends on retry (INC-4471, still open with vendor).
-- Remove when their fix ships.
where vendor_id != 447
```

**Still needed, but compensating for something fixable** ⇒ port it now, open a
ticket, note it in the model. Don't fix upstream data quality in the same change
as a conversion — that's [K11](../K-antipatterns/K11-convert-and-optimise.md) and it makes
failures impossible to attribute.

**No longer needed** ⇒ drop it, and expect a diff. This is the case that makes
[H11](../H-verification/H11-differences-that-should-exist.md) necessary: your new output will
legitimately differ from the old, and you need to be able to say why before
someone flags it as a conversion bug.

## The conversation

Most of these can't be resolved by reading. Take the list to whoever owns the
script and ask, per item: **what happened that made you add this?**

You'll get one of three answers — a specific incident, a vague memory, or nothing
at all. All three are useful. "Nobody knows" means it goes on the list of things
to test removing, carefully, after the conversion has landed.

## Write it down

In the [A9](A9-correctness-baseline.md) baseline, because it's the context that
makes a later diff interpretable:

```
compensating logic found:
  - `vendor_id not in (447, 892)`  → 447 confirmed still broken (INC-4471).
                                     892 unknown, nobody remembers. KEEP BOTH,
                                     retest 892 after cutover.
  - `qualify row_number() ...`     → dedups a 2023 double-load. Source is clean
                                     now. DROPPING — expect ~40 extra rows in
                                     2023 partitions vs baseline.
```

That second entry turns a day of diff-chasing into an expected result.

---

Previous: [A5 · Find the hidden state](A5-hidden-state.md) ·
Next: [A8 · Estimate conversion risk](A8-estimate-risk.md)
