# 7. Schema changes

Sources:
`dbt-bigquery/src/dbt/include/bigquery/macros/materializations/models/incremental/on_schema_change.sql`
and the generic equivalent in `dbt-adapters/.../models/incremental/on_schema_change.sql`.

`on_schema_change` defaults to **`'ignore'`**, set in the materialization:

```jinja
{% set on_schema_change = incremental_validate_on_schema_change(
       config.get('on_schema_change'), default='ignore') %}
```

## Changing it changes execution, not just schema handling

Repeating the point from [page 1](01-how-the-materialization-runs.md#when-a-temp-table-is-created-and-when-it-isnt)
because it is the part that bites:

```jinja
{% if on_schema_change != 'ignore' or language == 'python' %}
  {%- call statement('create_tmp_relation', language=language) -%}
    {{ bq_create_table_as(partition_by, True, tmp_relation, compiled_code, language) }}
  {%- endcall -%}
  {% set tmp_relation_exists = true %}
```

Setting `on_schema_change` to anything other than `ignore` forces a temp table to
be created before the main statement. For a `merge` model, that converts one
inlined statement into two: a `CREATE TABLE` followed by a `MERGE` reading from
it.

This is not free, and it is not documented as a performance decision. Your model
SQL now executes and materialises in full before the merge begins.

## The four values

| Value | Behaviour |
| --- | --- |
| `ignore` (default) | No comparison, no temp table. New columns in your model are silently dropped on write. |
| `fail` | Compare; raise if anything differs. |
| `append_new_columns` | Add columns present in source but not target. Never removes, never retypes. |
| `sync_all_columns` | Add, **remove**, and retype columns to match the source. |

`sync_all_columns` issues `alter_relation_add_remove_columns` and then
`alter_column_type` per changed type. It will drop target columns that your model
no longer produces — data loss by design, so be sure that's what you want.

## What counts as a change

`bigquery__check_for_schema_changes` sets `schema_changed = True` if any of:

- `source_not_in_target` is non-empty — columns you added,
- `target_not_in_source` is non-empty — columns you removed,
- `new_target_types` is non-empty — columns whose type changed.

Column comparison is by **name only**. From `diff_columns` in
`column_helpers.sql`, with the comment saying so explicitly: this "does not
perform a data type check". Type changes are detected separately by
`diff_column_data_types`, which flags a column only when
`sc.expanded_data_type != tc.expanded_data_type` **and**
`not sc.can_expand_to(other_column=tc)`.

That second condition is why widening is quiet. If the existing type can absorb
the new one, it isn't reported as a change and no `alter` is emitted.

## The `fail` message

Worth knowing verbatim, because it's the one people hit in CI:

```
The source and target schemas on this incremental model are out of sync!
They can be reconciled in several ways:
  - set the `on_schema_change` config to either append_new_columns or sync_all_columns...
  - Re-run the incremental model with `full_refresh: True` to update the target schema.
  - update the schema manually and re-run the process.
```

It then prints `source_not_in_target`, `target_not_in_source`, and
`new_target_types`, which is usually enough to diagnose without re-running.

## The BigQuery-specific part: `STRUCT` columns

This is where BigQuery diverges from every other adapter, and the reason it
overrides these macros at all. The file's header comment says it "augments the
core macro with STRUCT column synchronization logic".

In `bigquery__sync_column_schemas`:

```jinja
{% set struct_sync_result = adapter.sync_struct_columns(
    on_schema_change, source_relation, target_relation, schema_changes_dict) %}
{% if struct_sync_result is not none %}
  {% set struct_sync_dict = struct_sync_result %}
{% endif %}
```

`sync_struct_columns` (Python, in the adapter) is given the chance to rewrite the
add/remove/retype lists before any DDL runs. The generic macro has no equivalent
step.

The reason: BigQuery `STRUCT` columns are nested. Adding a field inside a
`STRUCT` is not adding a top-level column — it's a type change on the parent.
Naive column diffing would either miss it or try to add a column named
`parent.child`. The struct sync pass exists to translate nested changes into
operations BigQuery will accept.

Practical consequences:

- Nested-field changes are handled, but through a Python path that the logged
  add/remove/retype lists reflect only after rewriting.
- The `source_relation` is threaded through the `changes_dict` specifically so
  this hook can reach it — it's added, used, then `pop`ped back off at the end.

## What gets logged

Every sync logs:

```
In <target_relation>:
    Schema change approach: <on_schema_change>
    Columns added: [...]
    Columns removed: [...]
    Data types changed: [...]
```

Check this in your run logs after a schema change — particularly with
`sync_all_columns`, where "Columns removed" is the line that tells you whether
you just dropped a column you cared about.

## Return value

`bigquery__process_schema_changes` returns `schema_changes_dict['source_columns']`
— the temp table's columns — which the materialization then uses as
`dest_columns` for the merge. With `on_schema_change = 'ignore'` it returns `{}`,
and the materialization falls back to:

```jinja
{% if not dest_columns %}
  {% set dest_columns = adapter.get_columns_in_relation(existing_relation) %}
{% endif %}
```

So under `ignore`, the column list comes from the **existing target**. Any column
your model newly produces is simply not in the insert list, and is dropped
without comment. That's the mechanism behind "my new column isn't appearing" —
the most common incremental-model support question there is.

---

Previous: [6. `partition_by` in detail](06-partition-config.md) ·
Next: [8. Gotchas](08-gotchas.md)
