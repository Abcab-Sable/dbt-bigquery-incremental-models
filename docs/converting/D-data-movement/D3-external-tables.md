# D3 · External tables and BigLake

> **Part D — Data movement, DDL, metadata** · Sourcing: `CRAFT`
> **The question:** my script queries files in GCS directly. How does dbt see that?

As a source. External tables are inputs dbt doesn't build, which is the definition
of a source — [E3](../E-translation/E3-ref-vs-source.md).

## Declare it as a source

```yaml
sources:
  - name: lake
    schema: external
    tables:
      - name: raw_events
        description: External table over gs://acme-lake/events/, Parquet, hive-partitioned by dt.
```

```sql
select * from {{ source('lake', 'raw_events') }}
```

dbt doesn't need to know it's external — it's a relation it reads and doesn't
build. Everything downstream gets normal lineage.

## Creating them

dbt core doesn't create external tables. Options:

**`dbt-external-tables`** — a package that adds `dbt run-operation
stage_external_sources`, driven from your source YAML. Keeps the definition next
to everything else.

**Terraform or `bq mk`** — right if you already manage infrastructure that way.
The source YAML then documents something managed elsewhere, which is fine as long
as it's stated.

Either way, this is a deployment prerequisite, not part of the DAG. A model
referencing an external table that hasn't been created fails with a confusing
"not found" — worth noting in the source description.

## The cost model is different

This is the part that catches people converting a script.

External tables are **read from GCS on every query**. There's no BigQuery storage,
no cached statistics, and crucially **no partition pruning** unless the data is
hive-partitioned and the query filters on the partition key.

So a model reading an external table repeatedly is re-reading GCS repeatedly. If
several models read the same external table, materialise it once:

```sql
-- models/staging/stg_raw_events.sql
{{ config(materialized='table') }}
select * from {{ source('lake', 'raw_events') }}
```

Then everything downstream reads the BigQuery table. One GCS read per run instead
of one per consumer.

If the script read the external table once and wrote a native table, it was
already doing this — preserve that, don't "simplify" it into direct reads
everywhere.

## Hive partitioning

If the files are laid out as `gs://bucket/events/dt=2026-08-31/...`, the external
table can expose `dt` as a pseudo-column and prune on it. Filter on it explicitly:

```sql
select * from {{ source('lake', 'raw_events') }}
where dt >= '2026-08-01'
```

Without that filter you read the entire bucket. Same
[pruning discipline](../../beginner/04-partitioning-explained.md#what-your-query-prunes-actually-requires)
as native partitioning, different mechanism.

## Schema drift

External table schemas are either declared or inferred. Inferred means a new
column in the files appears silently, and a changed type can start erroring
mid-query rather than at load.

The script may have had an explicit schema that acted as a contract. If you're
moving to inference, add tests or a contract on the first downstream model to
recover the assertion — same argument as [D1](D1-load-data.md).

## BigLake specifics

BigLake tables add fine-grained access control over the same idea. From dbt's
perspective nothing changes — still a source. But the **permissions** differ: the
dbt service account needs access through the BigLake connection, not just to the
bucket.

This is a common "worked in dev, failed in prod" cause, alongside
[cross-project permissions](../E-translation/E4-cross-project-references.md).
Check before cutover.

---

Previous: [D2 · `EXPORT DATA` to GCS](D2-export-data.md) ·
Next: [D4 · Wildcard tables and `_TABLE_SUFFIX`](D4-wildcard-tables.md)
