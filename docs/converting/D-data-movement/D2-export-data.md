# D2 · `EXPORT DATA` to GCS

> **Part D — Data movement, DDL, metadata** · Sourcing: `CRAFT`
> **The question:** my script exports results to a bucket. Post-hook?

You can, and it's one of the more defensible post-hooks. But it's still
orchestration, and the better shape is usually a model plus a separate export
step.

## The pattern

```sql
EXPORT DATA OPTIONS(
    uri = 'gs://acme-exports/daily/*.csv',
    format = 'CSV',
    overwrite = true
) AS
SELECT * FROM analytics.daily_summary WHERE report_date = CURRENT_DATE();
```

Two things tangled: a query, and a delivery mechanism.

## Split them

**The query becomes a model.** It's a defined result set, so it deserves lineage,
tests, and a name:

```sql
-- models/exports/daily_summary_export.sql
{{ config(materialized='table') }}

select * from {{ ref('daily_summary') }}
where report_date = current_date()
```

**The export becomes a step after dbt.** Your orchestrator already runs dbt; it
can run one more thing:

```bash
dbt build --select daily_summary_export
bq extract --destination_format=CSV \
    'analytics.daily_summary_export' \
    'gs://acme-exports/daily/*.csv'
```

Now the data is inspectable before it ships, the export is visible in your
pipeline, and a failure tells you which half broke.

## If it has to be a post-hook

Sometimes the orchestrator isn't yours to change, or atomicity with the model
matters. Then:

```sql
{{ config(
    materialized='table',
    post_hook="""
        export data options(
            uri = 'gs://acme-exports/daily/{{ run_started_at.strftime('%Y%m%d') }}/*.csv',
            format = 'CSV',
            overwrite = true
        ) as select * from {{ this }}
    """
) }}
```

Know what you're accepting:

- **It runs on every build**, including dev, CI, and backfills — unless you gate
  it on `target.name`, which you should ([F3](../F-hooks/F3-empty-hook-skipping.md))
- **It runs before `apply_grants`** — [F4](../F-hooks/F4-where-hooks-run.md)
- **It doesn't run if the model fails** — [F16](../F-hooks/F16-hooks-and-failure.md)
- **There's no rollback.** Export succeeds, something later fails, the file is
  already in the bucket and possibly already consumed

Gate it, always:

```sql
post_hook="{% if target.name == 'prod' %} export data options(...) as select * from {{ this }} {% endif %}"
```

Without that gate, every developer's local run writes to the production bucket.

## `overwrite = true` and partial writes

`EXPORT DATA` with `overwrite` clears the destination prefix first. A failure
part-way through leaves the old files gone and the new ones incomplete.

If a downstream consumer polls that prefix, they can read a partial export. The
usual mitigation is exporting to a dated prefix and having consumers read the
latest complete one — which is why the example above includes
`run_started_at`.

## Wildcards produce many files

`gs://.../*.csv` produces one file per output shard, and the count isn't
predictable. If a consumer expects exactly one file, that's a fragile contract —
either they handle multiple, or you add a step to consolidate.

Worth checking what the script's consumer actually does before you change
anything, because this is a behaviour people depend on without documenting.

## Record it as a consumer

However you do the export, the destination is a **downstream consumer** of the
model. Note it in the model's YAML — an `exposure` is the formal way:

```yaml
exposures:
  - name: daily_export_to_partner
    type: application
    depends_on: [ref('daily_summary_export')]
    owner: {name: Data Platform, email: data@acme.com}
```

That's the thing that stops someone changing the model's columns without
realising a partner receives them — and it's exactly what
[I5](../I-migration/I5-notifying-consumers.md) needs at cutover.

---

Previous: [D1 · `LOAD DATA` from GCS](D1-load-data.md) ·
Next: [D3 · External tables and BigLake](D3-external-tables.md)
