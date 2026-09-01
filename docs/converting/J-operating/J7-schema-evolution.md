# J7 · Schema evolution over time

> **Part J — Operating it afterwards** · Sourcing: `SRC`
> **The question:** the model needs a new column next month. Then what?

Depends on `on_schema_change`, and on whether you set it during the conversion.
The default silently discards new columns, which is the single most common
post-conversion surprise.

## The immediate answer

```sql
{{ config(on_schema_change='append_new_columns') }}
```

Add the column to your `select`, run, and it appears. Without this — under the
default `ignore` — dbt takes the column list from the **existing table** and your
new column is dropped without comment
([D8](../D-data-movement/D8-add-column-migrations.md)).

## What each value gives you long-term

| Value | New columns | Removed columns | Type changes |
| --- | --- | --- | --- |
| `ignore` | Silently dropped | Retained | Ignored |
| `fail` | Run stops | Run stops | Run stops |
| `append_new_columns` | Added | Retained | **Detected, not applied** |
| `sync_all_columns` | Added | **Dropped** | Applied |

`append_new_columns` is the right default for most converted models. It handles
the common case and never destroys anything.

`sync_all_columns` drops columns your model stops producing — data loss by design.
Choose it only when you want the table to track the model exactly, and know that a
typo in a column name becomes a dropped column.

## The recurring costs

**It changes your execution plan.** Anything other than `ignore` forces a temp
table before the main statement — a `merge` model becomes `CREATE` plus `MERGE`,
with your model output materialised in full first. Permanent, not just on the run
that changes the schema
([the balanced track](../../balanced/01-how-the-materialization-runs.md#when-a-temp-table-is-created-and-when-it-isnt)).

**Widening is silent.** `diff_column_data_types` only flags a change when the
types differ **and** `can_expand_to` is false. A column quietly widening produces
no `alter` and no log line.

**Backfilled columns are null for history.** Adding a column populates it going
forward only. Existing partitions keep nulls until you rebuild them —
[G7](../G-scheduling/G7-backfill-partition-ranges.md).

That last one catches people: the column appears, looks broken for all historical
data, and the fix is a backfill nobody planned for.

## Check the log after a schema change

```
In analytics.daily_events:
    Schema change approach: append_new_columns
    Columns added: [device_type]
    Columns removed: []
    Data types changed: []
```

Read it after the first run following a model change — especially "Columns
removed" if you're on `sync_all_columns`.

## Enforce the shape where it matters

For models with real consumers, a contract turns drift into a build failure:

```yaml
models:
  - name: daily_events
    config:
      contract:
        enforced: true
    columns:
      - name: event_date
        data_type: date
      - name: event_count
        data_type: int64
```

This is the modern replacement for the script's explicit `CREATE TABLE` schema,
and it's better because it's checked every run rather than at creation.

## The gradual-change discipline

For a model with downstream consumers, the safe sequence is:

1. Add the new column ⇒ additive, breaks nothing
2. Backfill history if needed
3. Let consumers migrate
4. Remove the old column, in a separate change, after they have

Which is the same shape as [I2](../I-migration/I2-strangler-pattern.md), applied
to a schema instead of a project. Doing steps 1 and 4 together is how a routine
column rename breaks three dashboards.

---

Previous: [J6 · Freshness checks](J6-freshness-checks.md) ·
Next: [J8 · Growing partition counts](J8-partition-growth.md)
