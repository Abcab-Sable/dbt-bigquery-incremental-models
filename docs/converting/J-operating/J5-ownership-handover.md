# J5 · Ownership and on-call handover

> **Part J — Operating it afterwards** · Sourcing: `CRAFT`
> **The question:** who owns this now, and what do they need to know?

Whoever owned the script, unless you've agreed otherwise. Conversion changes the
tooling, not the accountability — and the handover is where a lot of converted
models quietly become orphans.

## Record the owner in the project

```yaml
models:
  - name: daily_events
    config:
      group: analytics_platform
    description: ...
```

```yaml
groups:
  - name: analytics_platform
    owner:
      name: Analytics Platform
      email: analytics-platform@acme.com
      slack: "#analytics-platform"
```

A group rather than an individual — individuals leave, and an unowned model is
one nobody can sign off changes to
([H13](../H-verification/H13-sign-off.md)).

## What on-call actually needs

Not the conversion story. Four things:

**How to rerun it.**

```bash
dbt build --select +daily_events
```

**How to backfill it**, with the exact command —
[G7](../G-scheduling/G7-backfill-partition-ranges.md).

**What the expensive operations are.** A full refresh scanning 2 TB should carry a
warning in the runbook, not be discovered at 3am
([G5](../G-scheduling/G5-backfill-full-refresh.md)).

**The model-specific failure mode.** Every incremental model has one. For a
dynamic `insert_overwrite` model it's the empty partition; for a `merge` model
with a predicate window it's duplicates from late updates. Name it explicitly:

```
Watch for: this model uses static partitions because refunds-only days
produce zero rows. If someone changes it to dynamic, empty days will
silently keep stale data. See B14.
```

That paragraph is the single most valuable line in the runbook.

## Hand over the knowledge, not just the model

The person who converted it knows things that aren't in the code:

- Which differences from the old script were deliberate
- Which compensating hacks were kept and why
- What the lookback window is sized for
- Which consumers were notified and agreed to what

All of that is in the baseline document
([I10](../I-migration/I10-documenting-decisions.md)) if you wrote it. Walk the new
owner through it once — twenty minutes now saves an afternoon later.

## The orphan risk

The specific failure: a migration team converts forty scripts, the project ends,
and nobody picks up ongoing ownership. Six months later something drifts and there
is nobody to notice.

Guard against it:

- Every model in a group with a real owner before the migration closes
- The [reconciliation job](J3-scheduled-reconciliation.md) owned by someone, with
  its output going somewhere a human looks
- Alerts routed to a team, not to the migration project's channel
- The migration channel **archived** rather than left as the default destination

The third is the one that catches people — alerts continue to fire into a channel
everyone has muted.

## Update it when the owner changes

Model ownership drifts as teams reorganise. A `group` in version control is at
least reviewable, which a wiki page isn't. Make it part of the reorg checklist
rather than something rediscovered during an incident.

---

Previous: [J4 · Alerting](J4-alerting.md) ·
Next: [J6 · Freshness checks](J6-freshness-checks.md)
