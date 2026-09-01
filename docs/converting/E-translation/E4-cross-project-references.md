# E4 · Cross-project and cross-dataset references

> **Part E — Statement-level translation** · Sourcing: `CRAFT`
> **The question:** my script reads from three projects. How does that survive?

BigQuery names things `project.dataset.table`, and scripts hardcode all three
parts. dbt generates them, which is what makes dev and prod separable — and what
breaks if you leave the hardcoding in.

## Reading from another project

Another project's table is external to your dbt project, so it's a
[source](E3-ref-vs-source.md), and the project goes in the source definition:

```yaml
sources:
  - name: billing
    database: acme-finance-prod      # BigQuery project
    schema: billing_raw              # BigQuery dataset
    tables:
      - name: invoices
```

```sql
select * from {{ source('billing', 'invoices') }}
```

Note the naming: dbt says **`database`** for what BigQuery calls a *project*, and
**`schema`** for what BigQuery calls a *dataset*. The mapping is stable, just
unintuitive the first time.

## Why this matters more than tidiness

A hardcoded `acme-finance-prod.billing_raw.invoices` reads **production** from
every environment. Your dev run, your CI run, and someone's local experiment all
hit prod.

Usually that means unexpected cost and a confusing lineage graph. Occasionally it
means a dev run writing somewhere it shouldn't. Either way, the fix is that the
project name belongs in configuration, not in SQL.

Point the source at different projects per target:

```yaml
sources:
  - name: billing
    database: "{{ 'acme-finance-prod' if target.name == 'prod' else 'acme-finance-dev' }}"
    schema: billing_raw
    tables:
      - name: invoices
```

## Writing to another project

Rarer, and usually worth questioning — but supported per model:

```sql
{{ config(
    materialized='incremental',
    database='acme-reporting',
    schema='published'
) }}
```

Before doing this, check the reason. "Finance needs it in their project" is often
better served by an authorized view than by writing across a boundary, and a view
doesn't need dbt to hold write permission in someone else's project.

## The permissions failure

Cross-project reads and writes need the dbt service account to have access in
*both* projects. Your script probably ran as a different principal — often a
person's account, or a service account with broader grants than you'd provision
now.

Check before you cut over, not after:

```sql
select user_email, count(*) as jobs
from `region-eu`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
where creation_time > timestamp_sub(current_timestamp(), interval 30 day)
  and query like '%acme-finance-prod%'
group by 1;
```

If that returns a principal that isn't your dbt service account, you have a
permissions task in front of the conversion. This is one of the most common
"converted fine in dev, failed in prod" causes.

## Regions

A query cannot join tables across BigQuery regions. If your script reads EU and
US datasets, something is copying data between them — find it, because it's
either a step you missed in [A1](../A-assess/A1-inventory.md) or a scheduled
transfer that's now an undeclared dependency.

---

Previous: [E3 · `ref` vs `source`](E3-ref-vs-source.md) ·
Next: [E5 · Finding every hardcoded table name](E5-finding-hardcoded-names.md)
