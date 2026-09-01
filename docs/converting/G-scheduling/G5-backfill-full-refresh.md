# G5 · Backfill via `--full-refresh`

> **Part G — Scheduling, parameters, backfills** · Sourcing: `SRC`
> **The question:** how do I rebuild the whole thing?

```bash
dbt run --select daily_events --full-refresh
```

Simplest backfill there is, and the right answer more often than people assume.
Two things to know before running it on something large.

## What it actually does

`should_full_refresh()` becomes true, so the materialization takes the
full-refresh branch:

```jinja
{% elif full_refresh_mode %}
    {% if not adapter.is_replaceable(existing_relation, partition_by, cluster_by) %}
        {% do log("Hard refreshing " ~ existing_relation ~ " because it is not replaceable") %}
        {{ adapter.drop_relation(existing_relation) }}
    {% endif %}
    {%- call statement('main', language=language) -%}
      {{ bq_create_table_as(partition_by, False, target_relation, compiled_code, language) }}
    {%- endcall -%}
```

Also note `is_incremental()` returns **false** during a full refresh, so your
`{% if is_incremental() %}` filter doesn't apply — the model reads everything.
That's the point, and it's also why the cost is different.

## Warning 1: it can drop the table

`is_replaceable` compares the existing table's partitioning and clustering against
your config. On a mismatch dbt **drops** before creating, logging
`Hard refreshing <relation> because it is not replaceable`.

So the first full refresh after changing `partition_by` or `cluster_by` is
destructive:

- there's a window where the table doesn't exist
- anything attached to the old table is gone — including
  [policy tags and row access policies](../D-data-movement/D11-policy-tags-rls.md)
- grants are reapplied afterwards by `apply_grants`; other metadata isn't

Plan for it. Don't discover it.

## Warning 2: cost

The whole point of the incremental model was not reading everything. A full
refresh reads everything, and on a large table that can be a very large single
query — or one that doesn't finish.

Check the expected scan before running it:

```bash
dbt compile --select daily_events --full-refresh
```

Then dry-run the compiled SQL:

```bash
bq query --dry_run --use_legacy_sql=false < target/compiled/.../daily_events.sql
```

If the number is uncomfortable, use [G6](G6-backfill-microbatch.md) or
[G7](G7-backfill-partition-ranges.md) instead.

Also check your cap — a `maximum_bytes_billed` sized for steady-state incremental
runs will fail the full refresh
([E12](../E-translation/E12-cost-controls.md)).

## When it's the right choice

- The table is small enough to rebuild comfortably
- You changed logic in a way that affects all history
- You changed `partition_by` and have to recreate anyway
- The model has drifted and you want a known-good state
- It's a `table` model, where every run is a full rebuild regardless

## Protecting against accident

For expensive models, guard it:

```sql
{% if should_full_refresh() and target.name == 'prod' and not var('allow_full_refresh', false) %}
  {{ exceptions.raise_compiler_error("Full refresh of " ~ this ~ " is expensive. Pass --vars '{allow_full_refresh: true}' if you mean it.") }}
{% endif %}
```

Crude, and it has saved people a five-figure query.

## Do it in a scratch dataset first

```bash
dbt run --select daily_events --full-refresh --target scratch
```

Gives you the cost, the runtime, and — usefully — a table to diff against
production. That diff is exactly
[J3](../J-operating/J3-scheduled-reconciliation.md)'s reconciliation, and running
it once during conversion tells you whether your incremental model has already
drifted.

---

Previous: [G4 · Environment variables and secrets](G4-env-vars-secrets.md) ·
Next: [G6 · Backfill via microbatch](G6-backfill-microbatch.md)
