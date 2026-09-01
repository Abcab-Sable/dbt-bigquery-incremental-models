# 4. The `insert_overwrite` strategy

Source: `dbt-bigquery/src/dbt/include/bigquery/macros/materializations/incremental_strategy/insert_overwrite.sql`

`insert_overwrite` replaces whole partitions. `partition_by` is mandatory —
without it, `bq_generate_incremental_insert_overwrite_build_sql` raises
*The 'insert_overwrite' strategy requires the `partition_by` config.*

From there, `bq_insert_overwrite_sql` makes one decision:

```jinja
{% if partitions is not none and partitions != [] %}
    {{ bq_static_insert_overwrite_sql(...) }}   {# static  #}
{% else %}
    {{ bq_dynamic_insert_overwrite_sql(...) }}  {# dynamic #}
{% endif %}
```

Supply a non-empty `partitions` config and you get the static path. Omit it (or
pass an empty list) and you get the dynamic path. Two more forks come from
`copy_partitions`, giving four distinct code paths in total.

## Dynamic partitions (the common case)

dbt discovers at runtime which partitions your model produced, then overwrites
exactly those. The generated script:

```sql
-- generated script to merge partitions into <target>
declare dbt_partitions_for_replacement array<<partition data type>>;

-- 1. create a temp table with model data
create or replace table <tmp> ... as ( <your model sql> );

-- 2. define partitions to update
set (dbt_partitions_for_replacement) = (
    select as struct
        array_agg(distinct <partition field> IGNORE NULLS)
    from <tmp>
);

-- 3. run the merge statement
merge into <target> as DBT_INTERNAL_DEST
    using (select * from <tmp>) as DBT_INTERNAL_SOURCE
    on FALSE
when not matched by source
     and <partition expr rendered on DBT_INTERNAL_DEST> in unnest(dbt_partitions_for_replacement)
    then delete
when not matched then insert (<cols>) values (<cols>);

-- 4. clean up the temp table
drop table if exists <tmp>;
```

The mechanism is worth internalising. `on FALSE` guarantees nothing matches, so
there is no update branch at all. Deletion is driven by `when not matched by
source`, restricted to the partitions in the array. Insertion is unconditional.
The net effect is: *delete everything in the touched partitions, then insert the
new rows*.

Step 1 is skipped if the temp table already exists (because `on_schema_change`
forced it) — the script says `-- 1. temp table already exists, we used it to
check for schema changes`.

Note `IGNORE NULLS` in step 2. The source comment explains it is aligned to
`_dbt_max_partition`, which also ignores nulls. Consequences below.

## The empty-partition trap

This is the failure mode that catches teams, and it follows directly from the
mechanism above.

**Only partitions present in the temp table are touched.** The delete is bounded
by `in unnest(dbt_partitions_for_replacement)`, and that array is built by
`array_agg(distinct ...)` over the rows your model produced *this run*.

So if your model produces **zero rows** for a partition, that partition is not in
the array, the `when not matched by source ... then delete` never fires for it,
and **whatever was there before stays there**.

Concretely: yesterday your model wrote 1,000 rows to `2026-08-31`. Today the
upstream data for that day is retracted and your model correctly produces zero
rows for it. Dynamic `insert_overwrite` leaves all 1,000 stale rows in place.
Your table now disagrees with your source, and nothing in the run log hints at
it. The run is "successful".

Deletions do not propagate. `insert_overwrite` overwrites the partitions you
*wrote*; it has no notion of the partitions you *meant to empty*.

The same reasoning applies to `NULL` partition values. Because of `IGNORE NULLS`,
a `NULL` partition key is never added to the replacement array. Rows with a
`NULL` partition value will be inserted (the insert branch is unconditional) but
never replace anything — so they accumulate on every run.

### Working around it

**Use the static path with an explicit partition list.** If you tell dbt which
partitions to overwrite, it overwrites them whether or not your model produced
rows — the predicate is built from your literals, not from the data:

```sql
{{ config(
    materialized='incremental',
    incremental_strategy='insert_overwrite',
    partition_by={'field': 'event_date', 'data_type': 'date'},
    partitions=['date_sub(current_date(), interval 1 day)', 'current_date()']
) }}
```

Now yesterday's partition is emptied even when the model returns nothing for it.

**Or make the emptiness explicit.** Have the model emit a row per expected
partition — a spine joined to your data — so every partition you care about is
always represented in the temp table.

**Or full-refresh on a schedule** to reconcile drift, accepting the cost.

The trap is not a bug. It is what "overwrite the partitions I produced" means. It
only bites when you read it as "make the target match my model's output", which
is what `table` materialization does and `insert_overwrite` does not.

## Static partitions

You supply the partition list; dbt does not query for it.

```jinja
{% set predicate %}
    {{ partition_by.render_wrapped(alias='dbt_internal_dest') }} in (
        {{ partitions | join(', ') }}
    )
{% endset %}
```

Values are joined into the SQL **verbatim**. They are SQL expressions, not
string literals — `current_date()` and `date_sub(...)` work, but a bare date must
be quoted by you (`'2026-08-31'`). There is no escaping and no type checking; this
is direct interpolation, so never build a `partitions` list from untrusted input.

The static path emits a single `MERGE` with no `declare`/`set` preamble, and no
temp table unless one already existed. From the source comment, this path "excels
where using a fixed minimal partition list, no need to query source data for
partitions, and insert statements can be batched. this also keeps things
deterministic."

One asymmetry: `get_insert_overwrite_merge_sql` is called with
`include_sql_header = not tmp_relation_exists`. Your `sql_header` is prepended
only when the model SQL is inlined; when a temp table exists, the header was
already applied by `create_table_as`. This is the only place in the codebase
where `include_sql_header` is true — the generic macro says so in a comment.

## `copy_partitions`

Instead of a `MERGE`, dbt copies partitions with the BigQuery copy API. Both the
static and dynamic paths support it, and both end up in `bq_copy_partitions`,
which builds `table$SUFFIX` decorators and calls `adapter.copy_table(..., "table")`.

That materialization argument matters. In `impl.py`:

```python
if materialization == "incremental":
    write_disposition = WRITE_APPEND
elif materialization == "table":
    write_disposition = WRITE_TRUNCATE
```

`bq_copy_partitions` passes `"table"`, so each destination partition is
**truncated and replaced**, not appended to.

Partition decorators are formatted by granularity:

| `partition_by` | Decorator format | Example |
| --- | --- | --- |
| `data_type: int64` | the integer as text | `tbl$42` |
| `granularity: hour` | `%Y%m%d%H` | `tbl$2026083114` |
| `granularity: day` | `%Y%m%d` | `tbl$20260831` |
| `granularity: month` | `%Y%m` | `tbl$202608` |
| `granularity: year` | `%Y` | `tbl$2026` |

Restrictions and behaviours:

- **`copy_partitions` requires `insert_overwrite` or `microbatch`.** With `merge`
  it is a compiler error, raised before any SQL runs: *The 'copy_partitions'
  option requires the 'incremental_strategy' option to be set to
  'insert_overwrite' or 'microbatch'.* The source comment is blunt: we can't copy
  partitions with merge strategy.
- **A temp table is always created**, on both paths — there is nothing to copy
  from otherwise.
- **The dynamic + copy path queries for partitions** with
  `select distinct <render_wrapped()> from <tmp>`, then copies each one. Note this
  is a plain `select distinct`, without the `IGNORE NULLS` guard used in the
  `MERGE` path.
- **The static + copy path resolves your literals first** by running
  `select cast(partition_literal as timestamp) ... from unnest([...])`. Because it
  casts to `timestamp`, this path is time-partition shaped; the integer branch in
  `bq_copy_partitions` exists but is reached via the dynamic path.

The static + copy path is the one that escapes the empty-partition trap by
design. The source says so directly: it "will still copy (truncate) a partition
even if the incremental run produced no rows for that date, because the user
explicitly listed it in `partitions=`."

Copying is billed as a copy job rather than a query, which is why it's attractive
for large partition rebuilds. The trade-off is more API calls — one copy per
partition, in a Python loop — so a run touching hundreds of partitions will spend
real time in that loop.

## `_dbt_max_partition`

A scripting variable you can reference in your model SQL to get the highest
partition value currently in the target. From `common.sql`:

```jinja
{%- if '_dbt_max_partition' in compiled_code and language == 'sql' -%}
    declare _dbt_max_partition {{ partition_by.data_type_for_partition() }} default (
      select max({{ partition_by.field }}) from {{ this }}
      where {{ partition_by.field }} is not null
    );
{%- endif -%}
```

Three things follow, all of which surprise people:

1. **It is declared only if the literal string `_dbt_max_partition` appears in your
   compiled code.** A plain substring check. Mention it in a comment and you get a
   spurious declaration; build the name dynamically and you get
   `Undeclared query parameter` at runtime.
2. **SQL only.** The `language == 'sql'` guard means Python models never get it.
   The source carries a `TODO: revisit partitioning with python models`.
3. **It ignores nulls**, matching the `IGNORE NULLS` in the dynamic path.

It is declared by `bq_create_table_as`, so it is in scope for the statement that
creates the temp table — which is where you want it.

---

Previous: [3. The `merge` strategy](03-merge.md) ·
Next: [5. The `microbatch` strategy](05-microbatch.md)
