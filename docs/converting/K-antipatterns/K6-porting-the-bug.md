# K6 · Porting the bug faithfully

> **Part K — Anti-patterns** · Sourcing: `CRAFT`
> **The question:** I matched the script exactly. Isn't that the goal?

Only if the script was right. A perfect conversion of a broken script is a broken
model with a clean face on it — and now it has tests and lineage vouching for it.

## How it happens

You're aiming for equivalence ([H1](../H-verification/H1-what-correct-means.md)),
which is correct. Parity passes. Everyone signs off. What nobody checked is
whether the baseline was ever right.

The script's bugs are now:

- reproduced exactly
- endorsed by a passing parity check
- harder to spot, because the code looks deliberate
- covered by tests asserting the buggy behaviour

That last one is the sting. A `unique` test on a key the script deduped
incorrectly now enforces the wrong thing.

## What to look for

The signatures from [A6](../A-assess/A6-compensating-hacks.md):

**Dedup on data that should already be unique.** Usually compensating for the
script's own non-idempotency — an append with no key, re-run once. Convert to a
`unique_key` and the cause disappears; keep the dedup and you've preserved the
scar without the wound.

**Hardcoded exclusions.**

```sql
where vendor_id not in (447, 892)
  and event_date != '2024-11-03'
```

Each encodes an incident. Some are still live; some were fixed years ago.

**Defensive `COALESCE`** on columns that shouldn't be nullable — masking an
upstream problem rather than handling one.

**A window wider than needed** with no explanation. Might be late-arriving data
([G8](../G-scheduling/G8-late-arriving-data.md)), might be superstition.

**A `DELETE` that isn't part of the write pattern** — cleaning up something the
script itself created.

## The triage

**Still needed and understood** ⇒ port it, with a comment naming the reason and
the ticket. The comment is the deliverable.

**Still needed but compensating for something fixable** ⇒ port it now, open a
ticket, note it. Don't fix upstream data quality in the same change —
[K11](K11-convert-and-optimise.md).

**No longer needed** ⇒ drop it, and **predict the resulting difference** before
you compare ([H11](../H-verification/H11-differences-that-should-exist.md)).

## The conversation

Most of these can't be resolved by reading. Take the list to the owner and ask
per item: **what happened that made you add this?**

Three possible answers, all useful:

- A specific incident ⇒ check whether it's still live
- A vague memory ⇒ test removing it, carefully, after cutover
- Nothing at all ⇒ strongest candidate for removal, and the finding is that
  nobody knows

## Don't fix it during the conversion

Tempting — you're right there, you can see the bug. But then a diff could be the
conversion or the fix, and you can't tell which.

Convert faithfully, verify, cut over. **Then** fix, as its own change with its own
verification. Two changes, two diffs, two clear attributions.

The exception: a bug so severe that reproducing it is unacceptable. Then fix it,
document it loudly as an intended difference, and get it agreed before cutover —
[I5](../I-migration/I5-notifying-consumers.md).

## Record what you found

Even the things you kept:

```
Compensating logic in daily_events.sql:
  vendor_id not in (447, 892)
    447 — still broken (INC-4471, open with vendor). KEPT.
    892 — nobody remembers. KEPT for now; retest 2026-10 (DATA-2340).
  qualify row_number() over (partition by event_id ...)
    Deduped a 2023 double-load. Source verified clean. DROPPED.
    Expect ~40 extra rows/day Mar–Jun 2023.
```

That's the artefact that stops the next person either deleting something
load-bearing or preserving something obsolete for another five years.

---

Previous: [K5 · Imperative structure in Jinja](K5-imperative-jinja.md) ·
Next: [K7 · Over-parameterising with vars](K7-over-parameterising.md)
