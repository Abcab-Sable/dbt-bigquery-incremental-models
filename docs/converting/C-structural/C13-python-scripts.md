# C13 · Python script using the BigQuery client

> **Part C — Structural archetypes** · Sourcing: `CORE✓`
> **The question:** we have a pandas script that writes a table. Does dbt do Python?

Yes, but on BigQuery it runs on **Dataproc**, not in BigQuery — which is a bigger
commitment than most people expect. Often the better answer is that the script was
mostly SQL wearing a Python coat.

## First: is it actually Python work?

A large share of "Python scripts" are a SQL query, a `to_dataframe()`, some pandas
that could be SQL, and a write back:

```python
df = client.query("SELECT * FROM raw.orders").to_dataframe()
df['total'] = df['qty'] * df['price']
df = df[df['status'] == 'active']
df.to_gbq('analytics.orders_enriched', if_exists='replace')
```

That's a model:

```sql
{{ config(materialized='table') }}

select *, qty * price as total
from {{ source('raw', 'orders') }}
where status = 'active'
```

Convert it as SQL. You get lineage, tests, and no Dataproc.

**Check the pandas honestly.** Filters, joins, group-bys, window operations,
string handling, date arithmetic — all SQL. What isn't: ML inference, calls to
external APIs, libraries with no SQL equivalent, genuinely iterative algorithms.

## What dbt Python models actually run on

From `dbt-bigquery`'s `python_submissions.py`, there are two helpers:

```python
class ClusterDataprocHelper(_BigQueryPythonHelper):
    # requires dataproc_cluster_name in profile or config

class ServerlessDataProcHelper(_BigQueryPythonHelper):
    # Dataproc Serverless batches
```

So a BigQuery Python model is a **PySpark job on Dataproc**, reading and writing
BigQuery via the Spark connector (default jar
`gs://spark-lib/bigquery/spark-bigquery-with-dependencies_...`).

Practical consequences:

- You need **Dataproc** — a cluster, or serverless configured
- Cluster submission errors without `dataproc_cluster_name`: *Need to supply
  dataproc_cluster_name in profile or config to submit python job with cluster
  submission method*
- Your code runs as **PySpark**, not pandas-on-a-VM. A pandas script does not
  port unchanged
- Startup latency is real — this is not a fast path for small work
- Extra IAM, networking, and cost surface

That's the commitment. It's worth it for genuine Spark-scale work and rarely
worth it to avoid rewriting fifteen lines of pandas as SQL.

## If it stays Python

A dbt Python model looks like:

```python
def model(dbt, session):
    dbt.config(materialized="table")
    orders = dbt.ref("orders")
    return orders.filter(orders.status == "active")
```

`dbt.ref()` gives you a Spark DataFrame, so it participates in the DAG properly.

Constraints from the materialization
([balanced track](../../balanced/01-how-the-materialization-runs.md#python-models)):

- **Python always creates a temp table**, regardless of `on_schema_change`
- **`insert_overwrite` is not supported** — compiler error
- **`time_ingestion_partitioning` is not supported** — compiler error
- **`_dbt_max_partition` is never declared** — it's gated on `language == 'sql'`

So an incremental Python model is `merge` only. If the script's write pattern was
delete-insert, that shape isn't available and you need `unique_key` semantics
instead — [B8](../B-write-patterns/B8-merge-on-clause-to-unique-key.md).

## If it stays outside dbt

Entirely legitimate, and often correct — a script calling an external API, running
inference, or doing something Spark isn't a fit for.

Keep it where it is, and declare its output as a `source`
([E3](../E-translation/E3-ref-vs-source.md)). Downstream models get lineage and
freshness; you skip the Dataproc dependency. This is the
[A7](../A-assess/A7-what-not-to-convert.md) middle ground and it's the right
answer more often than the Python model is.

## Decision order

1. **Is the logic expressible in SQL?** ⇒ convert to a SQL model. Most cases.
2. **Is it Spark-scale, or genuinely needs Python libraries?** ⇒ a Python model,
   accepting Dataproc.
3. **Neither?** ⇒ leave it, source its output.

Don't pick 2 to avoid rewriting SQL. The infrastructure cost outlives the
rewrite by years.

---

Previous: [C12 · Procedures calling procedures](C12-nested-procedures.md) ·
Next: [C14 · Shell, `bq` CLI, and Airflow orchestration](C14-orchestration.md)
