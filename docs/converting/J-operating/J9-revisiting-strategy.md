# J9 · When to revisit the strategy choice

> **Part J — Operating it afterwards** · Sourcing: `CRAFT`
> **The question:** we picked `merge` a year ago. Is it still right?

Maybe not. The choice was made against the data as it was, and both the data and
your understanding of it have moved.

## The triggers

**Cost is climbing faster than volume.** A `merge` model scales with *target*
size, so as history accumulates the cost grows even if daily input is flat. That's
the signature of a model that should be `insert_overwrite`
([J1](J1-cost-after-conversion.md)).

**Reconciliation keeps finding drift.** If [J3](J3-scheduled-reconciliation.md)
flags the same thing repeatedly, the strategy may not fit the data. Recurring
empty-partition drift means static `partitions`, not more monitoring.

**Runs stopped finishing.** A single `insert_overwrite` that used to complete and
now times out wants [microbatch](../G-scheduling/G6-backfill-microbatch.md).

**The data changed shape.** Rows that never used to be updated now are — `merge`
with a `unique_key` instead of `insert_overwrite`. Or the reverse: an upsert
source became append-only, and `merge`'s cost is no longer buying anything.

**Backfills became routine.** If you backfill monthly, microbatch's per-batch
retry stops being a backfill tool and becomes the everyday shape.

**Someone changed a config without understanding it.** The specific one to check:
static `partitions` switched to dynamic "to simplify". That reintroduces
[B14](../B-write-patterns/B14-when-the-range-can-empty.md) silently. A comment in
the config is the defence ([I9](../I-migration/I9-what-to-keep.md)).

## Re-ask the three questions

The same ones from [A3](../A-assess/A3-classify-by-write-pattern.md), against
today's data rather than the script's:

1. **Does the model ever modify a row it wrote previously?**
   Yes ⇒ `merge` with a `unique_key`. No ⇒ `insert_overwrite` is available and
   usually cheaper.
2. **Is the write bounded by a partition range?**
   Yes ⇒ `insert_overwrite`.
3. **Can a period legitimately produce zero rows?**
   Yes ⇒ static `partitions`, not dynamic.

Answers drift. Question 3 in particular — a source that never had quiet days
acquires them when a region is added or a product is retired.

## Treat it as a conversion

Changing strategy is a behaviour change, so it earns the same treatment as the
original conversion, in miniature:

- A baseline before ([A9](../A-assess/A9-correctness-baseline.md))
- Parity checking after ([H2](../H-verification/H2-row-count-parity.md),
  [H3](../H-verification/H3-checksum-parity.md))
- The forced empty-partition test
- Predicted differences written down
  ([H11](../H-verification/H11-differences-that-should-exist.md))

The machinery already exists. Reuse it rather than treating a strategy switch as a
config tweak — it changes what the model does to your data.

## Watch for the destructive ones

Some changes force a drop and recreate:

- Changing `partition_by` or `cluster_by` —
  [D6](../D-data-movement/D6-partitioning-ddl.md)
- Which loses [policy tags and row access
  policies](../D-data-movement/D11-policy-tags-rls.md)

Changing `incremental_strategy` alone doesn't, as long as the partitioning stays
the same. Worth knowing which side of that line your change falls on before you
schedule it.

## Schedule the review

Annually, or when cost crosses a threshold you set. Put it in the model's
documentation so it's a planned activity rather than something triggered by an
incident:

```yaml
models:
  - name: daily_events
    description: >
      ...
      Strategy reviewed 2026-09. Static insert_overwrite: refunds-only days
      produce zero rows. Next review 2027-09, or if monthly cost exceeds £400.
```

That last clause is the useful part — it names the number that would change the
answer.

---

Previous: [J8 · Growing partition counts](J8-partition-growth.md) ·
Next: [K1 · The mega-model](../K-antipatterns/K1-mega-model.md)
