# D9 · Column type changes

> **Part D — Data movement, DDL, metadata** · Sourcing: `SRC`
> **The question:** my script alters column types. Does `on_schema_change` handle that?

Only under `sync_all_columns`, and only for changes BigQuery permits. Widening is
handled silently by doing nothing at all, which is worth understanding before you
rely on it.

## What dbt detects

`diff_column_data_types` flags a column only when **both** hold:

```python
if sc.expanded_data_type != tc.expanded_data_type and not sc.can_expand_to(other_column=tc):
    result.append({'column_name': tc.name, 'new_type': sc.expanded_data_type})
```

Types differ, **and** the source type cannot expand to the target type.

So if the existing column can already absorb the new values — the classic case
being a wider string — nothing is reported and no `alter` is emitted. The change
is real and invisible.

## What it does with a detected change

Under `sync_all_columns` only:

```jinja
{% if new_target_types != [] %}
  {% for ntt in new_target_types %}
    {% do alter_column_type(target_relation, ntt['column_name'], ntt['new_type']) %}
  {% endfor %}
{% endif %}
```

Under `append_new_columns`, type changes are **not** applied — that value only
adds columns. Under `ignore`, nothing is detected at all.

| `on_schema_change` | Type change handled? |
| --- | --- |
| `ignore` | No — not even detected |
| `fail` | Detected, run stops |
| `append_new_columns` | Detected, **not applied** |
| `sync_all_columns` | Applied |

`append_new_columns` catching but not applying type changes surprises people. If
your script altered types, that value isn't sufficient.

## BigQuery limits what's possible

Most narrowing and many conversions aren't allowed in place. `INT64` to `STRING`,
`STRING` to `DATE`, reducing `NUMERIC` precision — BigQuery will refuse, and
`alter_column_type` will fail.

When that happens the practical routes are:

**Full refresh.** Rebuild with the new type. Simplest, and the honest answer for
most type changes.

**A new column.** Add `amount_v2`, backfill, migrate consumers, drop the old one.
The safe route when the table is large or the change is risky, and it's what a
careful script would have done anyway.

**Cast in the model.** Often the type change isn't wanted at all — the model
started producing a different type by accident, and the fix is a cast to restore
the original.

That third case is worth checking first. A type change during conversion is more
often an unintended consequence of rewriting the SQL than a deliberate migration.

## Where conversions introduce type changes accidentally

**Losing an explicit `CREATE`.** A script declaring `NUMERIC(10,2)` becomes
whatever your `select` infers unless you cast —
[B2](../B-write-patterns/B2-create-if-not-exists.md).

**`SUM()` promotion.** Summing `INT64` gives `INT64`, but summing `NUMERIC`
gives `NUMERIC`, and a division can produce `FLOAT64`. Order of operations
matters — [H9](../H-verification/H9-reconciling-numeric-precision.md).

**`SAFE_CAST` introduced for robustness.** Turns errors into nulls, which is a
behaviour change ([H8](../H-verification/H8-reconciling-nulls.md)).

**Literal `NULL` without a cast.** Infers `INT64`. `cast(null as string)` is what
you meant.

Capture the original types in [A9](../A-assess/A9-correctness-baseline.md) and
compare after the first build:

```sql
select column_name, data_type
from `project.analytics.INFORMATION_SCHEMA.COLUMNS`
where table_name = 'daily_events'
order by ordinal_position;
```

## Lock it down with a contract

If types matter — and after a conversion they usually do — assert them rather than
hoping:

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

Now a type drifting fails at build time with a clear message, instead of
propagating into downstream casts and joins. This is the modern replacement for
the script's explicit `CREATE TABLE` schema, and it's strictly better because it's
checked every run.

---

Previous: [D8 · `ALTER TABLE ADD COLUMN` migrations](D8-add-column-migrations.md) ·
Next: [D10 · Grants and authorized views](D10-grants-authorized-views.md)
