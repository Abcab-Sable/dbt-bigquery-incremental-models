# 1. How the materialization runs

Source: `dbt-bigquery/src/dbt/include/bigquery/macros/materializations/incremental.sql`

BigQuery does **not** use dbt's default incremental materialization. It ships its
own, and the differences matter. The default materialization (in
`dbt-adapters/.../models/incremental/incremental.sql`) builds an intermediate
relation and swaps it via `rename_relation`. The BigQuery one never does — there
is no backup relation and no rename step.

## The branch order

The materialization picks exactly one of five paths. They are evaluated in this
order, and the first match wins.

```
1. copy_partitions set, but strategy is not insert_overwrite/microbatch
       -> compiler error, nothing runs

2. existing_relation is none
       -> CREATE TABLE AS (full build)

3. existing_relation.is_view
       -> DROP the view, then CREATE TABLE AS

4. full_refresh_mode
       -> if partition/cluster spec changed: DROP first
       -> CREATE TABLE AS

5. otherwise
       -> the incremental path (strategy macros run here)
```

Two things worth pulling out.

**The strategy is validated before any SQL runs.** `dbt_bigquery_validate_get_incremental_strategy`
is called near the top, so a typo in `incremental_strategy` fails at compile time
rather than after you've paid for a scan. Valid values are exactly `merge`,
`insert_overwrite`, and `microbatch`; anything else raises. When unset, the
default is **`merge`**.

**Replacing a view with a table is not atomic.** Branch 3 drops the view first,
with the comment in source reading: there's no way to atomically replace a view
with a table on BigQuery. Between the drop and the create, the relation does not
exist. If you're switching a model from `view` to `incremental`, downstream
readers can see a missing table.

## Full refresh can silently become a drop

Branch 4 doesn't just re-create the table. It first asks
`adapter.is_replaceable(existing_relation, partition_by, cluster_by)`
(`impl.py`). That returns `True` only when the existing table's partitioning
**and** clustering specs match the config. If either changed, dbt logs
`Hard refreshing <relation> because it is not replaceable` and issues a `DROP`
before the `CREATE`.

The practical consequence: changing `partition_by` or `cluster_by` on an existing
model turns your next `--full-refresh` into a drop-and-recreate. Grants are
re-applied afterwards by `apply_grants`, but anything else attached to the old
table — and any reader mid-query — is on its own.

`_partitions_match` compares field name and granularity (case-insensitively) for
time partitioning, and the range spec for range partitioning. Note that
`data_type` is *not* part of the time-partitioning comparison — only the field
and the granularity.

## When a temp table is created (and when it isn't)

This is the single most misunderstood part of the materialization, and it drives
both cost and behaviour.

In the incremental path, the temp table is created up front **only if**:

```jinja
{% if on_schema_change != 'ignore' or language == 'python' %}
```

`on_schema_change` defaults to `'ignore'`. So for a default-configured SQL model,
**no temp table is created at this point**. `tmp_relation_exists` stays `false`,
and your model's compiled SQL is passed down to the strategy macro to be inlined
directly as a subquery in the `MERGE`.

What each strategy then does with `tmp_relation_exists = false`:

| Strategy | Behaviour when no temp table exists yet |
| --- | --- |
| `merge` | Inlines your model SQL as the `USING (...)` subquery. The merge builder never creates one itself. |
| `insert_overwrite`, dynamic | Creates the temp table anyway — it has to, in order to compute the partition list. |
| `insert_overwrite`, static | Inlines your model SQL as the `USING (...)` subquery. |
| `insert_overwrite`, static + `copy_partitions` | Creates the temp table in a separate statement. |

So `merge` on a default config runs your model SQL **inside** the `MERGE`
statement, while dynamic `insert_overwrite` always materialises it first. That is
a real cost and concurrency difference, and it flips the moment you set
`on_schema_change` to anything other than `ignore`.

Setting `on_schema_change = 'append_new_columns'` on a `merge` model therefore
does more than enable column syncing — it changes the execution shape from one
statement to two.

## Python models

`supported_languages=['sql', 'python']`, but with restrictions enforced in source:

- Python models **always** create a temp table, regardless of `on_schema_change`.
- `insert_overwrite` + Python raises: *The 'insert_overwrite' strategy is not yet
  supported for python models.*
- `time_ingestion_partitioning` + Python raises: *Python models do not support
  ingestion time partitioning.*

## After the main statement

In order: `run_hooks(post_hooks)`, then `apply_grants` (revoking only when
`should_revoke` is true), then `persist_docs`, then — if a temp relation was
created — `adapter.drop_relation(tmp_relation)`.

Note that several strategy paths *also* emit their own `drop table if exists`
for the temp relation as part of the generated script. The cleanup is
belt-and-braces, not a single owner.

---

Next: [2. Choosing a strategy](02-choosing-a-strategy.md)
