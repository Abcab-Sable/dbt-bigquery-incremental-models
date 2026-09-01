# D1 · `LOAD DATA` from GCS

> **Part D — Data movement, DDL, metadata** · Sourcing: `CORE✓`
> **The question:** my script loads a CSV from GCS. Is that a model?

No. Loading is not transforming, and dbt has no `LOAD DATA` equivalent. It becomes
one of three things depending on what the file is.

## The three answers

**A seed**, if the file is small, static, and belongs in version control —
lookup tables, mappings, country codes.

**An external table or a `source`**, if the file is data that keeps arriving.

**Stays outside dbt**, if it's a genuine ingestion pipeline. Most cases.

## Seeds

Put the CSV in `seeds/` and run `dbt seed`:

```
seeds/country_codes.csv
```

```yaml
seeds:
  - name: country_codes
    config:
      column_types:
        iso_code: string
        population: int64
```

Then `{{ ref('country_codes') }}` like any model. Version-controlled, reviewable,
testable.

Seeds load in batches — `default__get_batch_size()` returns **10,000** rows per
insert. That tells you the intended scale: seeds are for hundreds or thousands of
rows, not millions. A 2 GB CSV as a seed will be slow and will bloat your repo.

**Rule:** if a human maintains it and it fits in a code review, it's a seed. If a
system produces it, it isn't.

## External tables

If files keep landing in GCS and you want to query them in place, define an
external table and treat it as a source. dbt doesn't create these natively —
`dbt-external-tables` does, or manage it with Terraform.

Either way, the resulting object is a `source`, not a model —
[D3](D3-external-tables.md).

## Leave the load where it is

The usual and correct outcome. Your loader — Fivetran, a Cloud Function, a
scheduled transfer, a script — keeps running. You declare its output:

```yaml
sources:
  - name: raw
    schema: raw
    tables:
      - name: events
        loaded_at_field: _loaded_at
        freshness:
          warn_after: {count: 2, period: hour}
          error_after: {count: 6, period: hour}
```

Now the load has a place in the lineage, and `dbt source freshness` tells you when
it stalls — which the script never did. That's most of the value of "converting"
the load, without converting it.

## Why loads shouldn't be models

Three reasons worth stating plainly:

**Not idempotent in the model sense.** A load appends what's in the bucket. Run it
twice and you get the data twice, unless the loader is careful. Models are
expected to be re-runnable — [E7](../E-translation/E7-idempotency-meaning.md).

**No dependencies to express.** A load's input is a file, which isn't in the DAG.
A model with no `ref()` and no `source()` is a root node that dbt can't reason
about.

**Failure modes are different.** Missing files, schema drift in the CSV, partial
uploads. dbt's model failure handling isn't shaped for these.

## The schema-drift trap

If the script did `LOAD DATA` with an explicit schema, that schema was a contract.
Moving to an external table or a source means BigQuery infers it, and an upstream
column change silently propagates.

Guard it with tests on the source, or a contract on the first model downstream:

```yaml
models:
  - name: stg_events
    config:
      contract:
        enforced: true
    columns:
      - name: event_id
        data_type: string
```

That gets you back the assertion the script's explicit schema was making.

---

Next: [D2 · `EXPORT DATA` to GCS](D2-export-data.md) ·
Back to [the backlog](../BACKLOG.md)
