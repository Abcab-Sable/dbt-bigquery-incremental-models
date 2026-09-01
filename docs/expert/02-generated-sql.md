# 2. Generated SQL

Emitted statements per path. `default__get_merge_sql` and
`default__get_insert_overwrite_merge_sql` live in
`dbt-adapters/.../models/incremental/merge.sql`.

## merge

```sql
{{ sql_header }}                                   -- if config'd, always included
merge into <target> as DBT_INTERNAL_DEST
    using ( <sql> | select [_PARTITIONTIME,] * from <tmp> ) as DBT_INTERNAL_SOURCE
    on (<pred>) and (<pred>) ...

when matched then update set                       -- omitted entirely if no unique_key
    <update_columns> = DBT_INTERNAL_SOURCE.<col>, ...

when not matched then insert
    (<dest_cols_csv>)
values
    (<dest_cols_csv>)
```

Predicate assembly order in `bq_generate_incremental_merge_build_sql`:

1. `incremental_predicates` (copied, not mutated in place)
2. `predicate_for_avoid_require_partition_filter()` if non-none
3. then in `default__get_merge_sql`: the unique-key predicate(s), or `'FALSE'`

No `unique_key` ⇒ `on (FALSE)` ⇒ unconditional insert. Append semantics, no
warning.

`update_columns` from `get_merge_update_columns`:
`merge_update_columns` (as-given) xor `merge_exclude_columns`
(case-insensitive, rebuilt from `dest_columns`, so properly quoted); both set ⇒
compiler error; neither ⇒ `dest_columns | map(attribute="quoted")`.

## insert_overwrite — dynamic

```sql
declare dbt_partitions_for_replacement array<{{ data_type_for_partition() }}>;

-- 1. create temp table (skipped if tmp_relation_exists)
{{ bq_create_table_as(partition_by, True, tmp_relation, sql, 'sql') }}

-- 2. define partitions to update
set (dbt_partitions_for_replacement) = (
    select as struct
        array_agg(distinct {{ partition_field }} IGNORE NULLS)
    from <tmp>
);

-- 3. run the merge statement
merge into <target> as DBT_INTERNAL_DEST
    using ( select [_PARTITIONTIME,] * from <tmp> ) as DBT_INTERNAL_SOURCE
    on FALSE
when not matched by source
     and {{ render_wrapped(alias='DBT_INTERNAL_DEST') }} in unnest(dbt_partitions_for_replacement)
    then delete
when not matched then insert (<cols>) values (<cols>);

-- 4. clean up
drop table if exists <tmp>
```

`partition_field` in step 2 is `time_partitioning_field()` under
`time_ingestion_partitioning`, else `render_wrapped()`.

`IGNORE NULLS` is stated in source as alignment with `_dbt_max_partition`.
Consequence: null-partition rows insert but never participate in replacement.

Replacement set is derived from **produced rows**. Zero rows for a partition ⇒
absent from array ⇒ no delete ⇒ prior contents retained.

## insert_overwrite — static

```sql
{{ sql_header }}                                   -- iff include_sql_header = not tmp_relation_exists
merge into <target> as DBT_INTERNAL_DEST
    using ( <sql> | select [...] * from <tmp> ) as DBT_INTERNAL_SOURCE
    on FALSE
when not matched by source
     and {{ render_wrapped(alias='dbt_internal_dest') }} in ( {{ partitions | join(', ') }} )
    then delete
when not matched then insert (<cols>) values (<cols>);

drop table if exists <tmp>;                        -- iff tmp_relation_exists
```

`partitions` values are interpolated verbatim — SQL expressions, not literals.
No escaping, no type check. This is the only call site where
`include_sql_header` is true.

Note the lowercase `dbt_internal_dest` alias here versus uppercase in the dynamic
path. Cosmetic; BigQuery identifiers are case-insensitive.

## insert_overwrite — copy_partitions

Both variants converge on `bq_copy_partitions` →
`adapter.copy_table(src, dst, "table")` → `WRITE_TRUNCATE` (`"incremental"` would
be `WRITE_APPEND`; it is not used here). Destination partitions are truncated and
replaced.

Dynamic variant resolves the set with:

```sql
select distinct {{ render_wrapped() }} from <tmp>
```

— plain `select distinct`, **without** the `IGNORE NULLS` guard used in the merge
path.

Static variant resolves literals first:

```sql
select cast(partition_literal as timestamp) as partition_ts
from unnest([ {{ partitions | join(', ') }} ]) as partition_literal
```

The `timestamp` cast makes this path time-partition shaped; `bq_copy_partitions`'
`int64` branch is reachable via the dynamic path.

Static + copy is the only path that truncates a listed partition when the run
produced no rows for it — stated explicitly in the source comment.

Decorator formats (`bq_copy_partitions`):

| Condition | Format | Example |
| --- | --- | --- |
| `data_type == 'int64'` | `\| as_text` | `t$42` |
| `granularity == 'hour'` | `%Y%m%d%H` | `t$2026083114` |
| `granularity == 'day'` | `%Y%m%d` | `t$20260831` |
| `granularity == 'month'` | `%Y%m` | `t$202608` |
| `granularity == 'year'` | `%Y` | `t$2026` |

One `copy_table` call per partition, in a Jinja `for` loop. Latency scales with
partition count.

## _dbt_max_partition

```sql
declare _dbt_max_partition {{ data_type_for_partition() }} default (
  select max({{ partition_by.field }}) from {{ this }}
  where {{ partition_by.field }} is not null
);
```

Emitted by `declare_dbt_max_partition`, called from `bq_create_table_as`, gated
on `'_dbt_max_partition' in compiled_code and language == 'sql'` — a literal
substring test. Dynamic name construction ⇒ never declared ⇒ `Undeclared query
parameter`. Mention in a comment ⇒ spurious declaration.

Uses the **raw** `partition_by.field`, not `render()`. On hourly timestamp
partitioning this yields the max timestamp, not the hour boundary.

## Ingestion-time rewrite

`wrap_with_time_ingestion_partitioning_sql`:

```sql
select TIMESTAMP({{ field }}) as _PARTITIONTIME, * EXCEPT({{ field }}) from ( <sql> )
```

Partition column is removed from output. `bq_create_table_as` runs `CREATE` then
`INSERT` rather than `CREATE TABLE AS` — ingestion-time tables cannot be created
with transformed data.

Asymmetry: `time_partitioning_field()` returns `_PARTITIONDATE` for
`data_type == 'date'`, but `insertable_time_partitioning_field()` always returns
`_PARTITIONTIME` ("Practically, only _PARTITIONTIME works so far").

---

Previous: [1. Control flow and dispatch](01-control-flow.md) ·
Next: [3. Semantics and edge cases](03-semantics.md)
