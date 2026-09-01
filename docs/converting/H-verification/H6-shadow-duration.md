# H6 · How long to shadow, and what ends it

> **Part H — Proving correctness** · Sourcing: `CRAFT`
> **The question:** when have I run both for long enough?

Long enough to have seen the cases that would break it. Decide that up front, in
cases rather than in days, or the period ends when someone loses patience.

## Cases, not days

"Two weeks" is arbitrary. What you actually need is to have observed:

- [ ] A **normal day** — parity clean
- [ ] A **quiet period** where some partition produced zero rows —
      [B14](../B-write-patterns/B14-when-the-range-can-empty.md)
- [ ] A **late-arriving batch**, if that happens to you
- [ ] A **re-run or retry** of the new model — [E8](../E-translation/E8-idempotency-proving.md)
- [ ] A **month boundary**, if any logic is month-shaped
- [ ] A **full weekly cycle**, if weekends differ from weekdays
- [ ] An **upstream failure or delay**, if you can afford to observe one

Most of these can be forced rather than waited for. Forcing is better — it's
faster and you control the timing.

In practice this lands at one to two weeks for a daily model, because a weekly
cycle is usually the binding constraint.

## What "clean" means

Define it before you start, or every difference becomes a debate:

```
Clean day = ops.shadow_parity has zero rows for that date,
            excluding 2026-08-14 (known-bad upstream, both sides differ).
```

Then: **N consecutive clean days**, where N covers your cycle. Not "mostly
clean" — a difference you decided to ignore is a difference you didn't explain,
and it's the one that turns out to matter.

## Restart the clock on changes

If you change the model during the shadow period, the evidence before the change
is about a different model.

You don't need the full period again for a comment or a rename. You do for
anything touching strategy, `unique_key`, `partition_by`, predicates, or the
model body. Note it:

```
2026-09-03  changed incremental_predicates window 7d → 14d. Clock restarted.
```

Without that line, a fortnight of "evidence" may be four days of the current model
and ten of its predecessor.

## Ending it early

Legitimate reasons:

- The model is trivial (`CREATE OR REPLACE` → `table`) and there's nothing
  time-dependent to observe
- You forced every case in the list within days
- Blast radius is low enough that finding out in production is acceptable —
  [A8](../A-assess/A8-estimate-risk.md)

Not legitimate: schedule pressure, or a run of clean days that hasn't covered
the cases. The empty-partition case in particular can hide for weeks and then
appear.

## Running it too long

There's a cost the other way. An indefinite shadow period means:

- Paying for everything twice
- Two implementations drifting as someone patches one and not the other
- Nobody sure which is authoritative

If you've hit your criteria, cut over. If you keep extending, the honest reading
is usually that you don't trust the criteria — fix those instead of adding weeks.

## Record the outcome

```
Shadow period: 2026-09-01 → 2026-09-14
Clean days:    13 of 14 (2026-09-08 known-bad upstream, both sides identical)
Cases seen:    normal ✓  empty partition ✓ (forced 09-05)  late arrival ✓ (09-11)
               re-run ✓ (forced 09-03)  month boundary — n/a  weekly cycle ✓
Verdict:       criteria met, proceed to cutover
```

That paragraph is the input to [H13](H13-sign-off.md), and the thing you'll want
in six months when someone asks how carefully this was checked.

---

Previous: [H5 · Shadow mode](H5-shadow-mode.md) ·
Next: [H7 · Reconciling: ordering](H7-reconciling-ordering.md)
