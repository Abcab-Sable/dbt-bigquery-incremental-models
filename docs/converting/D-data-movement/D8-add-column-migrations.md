# D8 · `ALTER TABLE ADD COLUMN` migrations

> **Part D — Data movement, DDL, metadata** · Sourcing: `SRC`
> **The question:** my script adds columns when they're missing. What replaces that?

`on_schema_change`. And the default is `ignore`, which silently discards new
columns — so this is a setting you must choose deliberately during conversion.

## The pattern

```sql
ALTER TABLE analytics.daily_events ADD COLUMN IF NOT EXISTS device_type STRING;

INSERT INTO analytics.daily_events (event_date, user_id, event_count, device_type)
SELECT ... ;
```

The `ALTER` is defensive — it makes deploys safe when the model gains a column.

## The replacement

```sql
{{ config(
    materialized='incremental',
    on_schema_change='append_new_columns'
) }}
```

Now adding a column to your `select` adds it to the table. The `ALTER` is gone.

## The default will bite you

`on_schema_change` defaults to **`ignore`**. Under `ignore`, dbt takes the column
list from the **existing table**:

```jinja
{% if not dest_columns %}
  {% set dest_columns = adapter.get_columns_in_relation(existing_relation) %}
{% endif %}
```

A column your model newly produces isn't in that list, so it isn't in the insert,
and it's dropped without comment. No error. This is the most common "why isn't my
change showing up" question there is.

If the script had an `ALTER ... ADD COLUMN IF NOT EXISTS`, it was explicitly
handling schema evolution — so leaving `on_schema_change` at its default
**removes a behaviour the script had**.

## The four values

| Value | Behaviour |
| --- | --- |
| `ignore` (default) | New columns silently discarded |
| `fail` | Run stops, listing the differences |
| `append_new_columns` | Adds new columns. Never removes, never retypes |
| `sync_all_columns` | Adds, **removes**, and retypes to match |

`append_new_columns` is the closest match to `ADD COLUMN IF NOT EXISTS`, and the
right default for a converted script.

`sync_all_columns` **drops** columns your model no longer produces. That's data
loss by design — only choose it if the script also dropped columns, which is rare.

## It changes your execution plan

The part that isn't about schemas at all:

```jinja
{% if on_schema_change != 'ignore' or language == 'python' %}
  {%- call statement('create_tmp_relation', language=language) -%}
```

Anything other than `ignore` **forces a temp table** before the main statement. A
`merge` model goes from one inlined statement to a `CREATE` plus a `MERGE`, and
your model's output now fully materialises before the merge begins.

So setting `on_schema_change` is a performance decision as well as a schema one.
Usually worth it — but measure if the model is large, and know why the timing
changed. See
[the balanced track](../../balanced/01-how-the-materialization-runs.md#when-a-temp-table-is-created-and-when-it-isnt).

## What counts as a change

`diff_columns` compares **names only** — the source comment says it "does not
perform a data type check". Type changes are detected separately by
`diff_column_data_types`, and only flagged when the types differ **and**
`can_expand_to` is false.

So widening is silent: no `alter` is emitted, and nothing reports it. See
[D9](D9-column-type-changes.md).

## Check the log

Every sync logs what it did:

```
In analytics.daily_events:
    Schema change approach: append_new_columns
    Columns added: [device_type]
    Columns removed: []
    Data types changed: []
```

Worth checking after the first run following a model change — especially the
"Columns removed" line if you chose `sync_all_columns`.

## The alternative: full refresh

For a one-off column addition, `dbt run --full-refresh` rebuilds the table with
the new shape and you never touch `on_schema_change`.

Fine for a rarely-changing model. Not fine as a routine answer — a full refresh on
a large incremental model is the cost you converted to avoid, and if the
partitioning changed it's a
[drop and recreate](../../balanced/01-how-the-materialization-runs.md#full-refresh-can-silently-become-a-drop).

---

Previous: [D7 · Expiration, labels, description](D7-table-options.md) ·
Next: [D9 · Column type changes](D9-column-type-changes.md)
