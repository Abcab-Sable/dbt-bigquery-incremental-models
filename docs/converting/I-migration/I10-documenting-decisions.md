# I10 · Documenting the conversion decisions

> **Part I — Migration strategy** · Sourcing: `CRAFT`
> **The question:** what do I write down, and where does it go?

Three artefacts, each in a different place because each has a different reader.
Written during the conversion, not after — nobody writes these retrospectively.

## 1. The model's own documentation

For anyone reading the model. Lives in the schema YAML, surfaces in the docs site:

```yaml
models:
  - name: daily_events
    description: >
      Daily event counts by user, one row per user per day.

      Converted from daily_events.sql (scheduled query, 03:00 UTC) in
      DATA-2288. Uses static insert_overwrite partitions because
      refunds-only days legitimately produce zero rows.
    columns:
      - name: event_date
        description: UTC date the events occurred on. Partition column.
        data_tests: [not_null]
      - name: user_id
        data_tests: [not_null]
```

Keep it about the data. The migration detail is one sentence; the rest belongs in
the baseline document.

## 2. The baseline document

For whoever investigates a discrepancy later. Committed next to the model —
[I9](I9-what-to-keep.md):

```markdown
# daily_events — conversion baseline

Converted 2026-09-09 from daily_events.sql. DATA-2288.

## What the script did
- Schedule: 03:00 UTC daily (scheduled query `daily_events_load`)
- Pattern: DELETE WHERE event_date >= start_date; INSERT SELECT
- Range: last 3 days
- Empty periods: DELETE was unconditional — empty days WERE emptied

## Baseline figures (2026-09-01)
- 412,883,201 rows, 1,096 partitions, 2024-09-01 → 2026-09-01
- Per-partition counts: analytics_baseline.daily_events_counts_20260901
- Snapshot: analytics_baseline.daily_events_20260901 (expires 2026-12-01)
- Typical run: 89 GB scanned, 47s

## Decisions
- Static `partitions`, not dynamic — the script emptied empty days and
  dynamic insert_overwrite would not. See B14.
- on_schema_change: append_new_columns — the script had
  ALTER TABLE ADD COLUMN IF NOT EXISTS.
- Dropped the 2023 QUALIFY dedup: source is clean now (verified).

## Known differences (all predicted before comparison)
1. Mar–Jun 2023: ~40 more rows/day. Dropped dedup. Legitimate.
2. 2026-08-14: differs both sides. Known-bad upstream, pre-existing.
3. `status` NULL where previously ''. Removed COALESCE.
   Downstream impact communicated; agreed with Priya 2026-08-30.

## Verification
- Column-level parity 2026-08-01..31, all columns except _loaded_at
- Shadow 2026-09-01..14, 13/14 clean days
- Forced empty-partition test passed 2026-09-05
- Two-run idempotency confirmed
- Signed off: Priya Raman, 2026-09-08
```

That's the document that answers "why is this model like this" and "did the
numbers change in 2023".

## 3. The runbook entry

For whoever is on call. Wherever your operational docs live:

```
daily_events
  Owner:     Data Platform
  Schedule:  02:00 UTC, tag:daily
  Rebuild:   dbt build --select +daily_events
  Backfill:  dbt run --select daily_events --vars '{backfill_start: ..., backfill_end: ...}'
  Full:      dbt run --select daily_events --full-refresh
             ⚠ ~2 TB scan. Check maximum_bytes_billed first.
  Watch for: stale partitions if a day produced no rows — check J3
             reconciliation output before assuming the model is fine.
```

Commands people need at 3am, and the one failure mode specific to this model.

## Write them as you go

The baseline's figures section must exist before you convert
([A9](../A-assess/A9-correctness-baseline.md)). The decisions section fills in as
you make them. The differences section is written **before** the first comparison —
that's what makes it a prediction rather than a rationalisation
([H11](../H-verification/H11-differences-that-should-exist.md)).

Assembled at the end, it's a document nobody quite remembers the details for.
Written as you go, it costs almost nothing.

## Keep the ticket honest

Whatever tracker you use, the ticket should end with the outcome rather than
"done":

```
Converted. Static partitions (B14 applies — refunds-only days).
3 known differences, all agreed. Rollback window to 2026-09-30.
Baseline: models/events/daily_events.baseline.md
```

The next person searching for "daily_events" finds that before they find the code.

---

Previous: [I9 · What to keep from the old script](I9-what-to-keep.md) ·
Next: [J1 · Cost after conversion](../J-operating/J1-cost-after-conversion.md)
