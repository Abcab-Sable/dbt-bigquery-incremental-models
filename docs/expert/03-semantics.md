# 3. Semantics and edge cases

## unique_key: three branches, two behaviours

```jinja
{% if unique_key is sequence and unique_key is not mapping and unique_key is not string %}
    {% for key in unique_key %}
        DBT_INTERNAL_SOURCE.{{ key }} = DBT_INTERNAL_DEST.{{ key }}   {# bare = #}
    {% endfor %}
{% else %}
    get_merge_unique_key_match(source_unique_key, target_unique_key)
{% endif %}
```

**The list branch never calls `get_merge_unique_key_match`.** Null-safe
comparison is unreachable for composite keys regardless of behaviour-flag state.
Nullable key components ⇒ perpetual non-match ⇒ duplicate insert per run.

Single-key branch, `bigquery__get_merge_unique_key_match`:

| `enable_truthy_nulls_equals_macro` | Emitted |
| --- | --- |
| off | `(src = tgt)` |
| on | `((src is null and tgt is null) or (src = tgt))` |

BigQuery deliberately does **not** use the generic `equals` macro's
`IS NOT DISTINCT FROM`. Per the source comment: inside a `MERGE` on a table with
`require_partition_filter=True`, that form causes the pruning analyzer to stop
recognising the `predicate_for_avoid_require_partition_filter` tautology as a
valid partition filter, and the merge **fails at runtime**. The longhand form is
semantically equivalent and preserves pruning.

## require_partition_filter predicate

`predicate_for_avoid_require_partition_filter(target='DBT_INTERNAL_DEST')`, gated
on `partition_config and config.get('require_partition_filter')`:

```sql
( `{{ target }}`.`{{ partition_field }}` is null
  or `{{ target }}`.`{{ partition_field }}` is not null )
```

Field is `time_partitioning_field()` under ingestion-time partitioning, else raw
`partition_config.field`. Tautological — satisfies the requirement, restricts
nothing, yields no pruning. Applied on the merge path only.

## Partition rendering

`_partition.py`. `parse()` lowercases all string values, including `field`.

`data_type_should_be_truncated()`:

```python
not (data_type == "int64" or (data_type == "date" and granularity == "day"))
```

`render(alias)` ⇒ `{data_type}_trunc({col}, {granularity})` when truncated, else
bare column (or `time_partitioning_field()` under ingestion-time).

`render_wrapped(alias)` adds two normalisations:

| Condition | Result |
| --- | --- |
| `int64` ∧ `range` set ∧ `interval ∉ (1,"1")` | `({col} - MOD({col} - {start}, {interval}))` |
| `data_type ∈ {date,timestamp,datetime}` ∧ not truncated ∧ not (ingestion ∧ date) | `{data_type}({render(alias)})` |
| else | `render(alias)` |

The second condition is satisfiable only for `date`+`day` (the other
non-truncated type is `int64`, excluded by the type set), yielding `date(col)`.

`int64` normalisation exists to collapse distinct *values* to distinct
*partitions* before `array_agg` — otherwise the replacement array is
cardinality-of-column rather than cardinality-of-partitions.

### Released-vs-main delta

The `interval ∉ (1,"1")` guard is present on `main` @ `e7553c7`, **absent in
released 1.12.0**. On 1.12.0, `interval: 1` emits `(col - MOD(col - start, 1))` —
arithmetically identity, but an expression rather than a bare column, which
defeats partition elimination. Main returns the raw column "to preserve partition
elimination".

This is the only behavioural difference between the released package and the
pinned commit. All incremental macro files are byte-identical.

## Rendering call sites

| Site | Expression |
| --- | --- |
| static i/o predicate | `render_wrapped(alias='dbt_internal_dest')` |
| dynamic i/o predicate | `render_wrapped(alias='DBT_INTERNAL_DEST')` |
| dynamic partition discovery | `render_wrapped()` |
| copy_partitions discovery | `render_wrapped()` |
| `require_partition_filter` guard | raw field / `time_partitioning_field()` |
| `_dbt_max_partition` | raw `partition_by.field` |

Both sides of the `in` comparison render identically, so static `partitions`
literals must be supplied **at partition granularity**. Monthly partitioning:
`'2026-08-15'` never matches `date_trunc(dest, month)`.

## data_type_for_partition

```python
return data_type if not time_ingestion_partitioning \
       else ("date" if data_type == "date" else "timestamp")
```

Types the `declare ... array<>`.

## microbatch validation

Reads the **raw** config dict, not the parsed `PartitionConfig`:

```jinja
{% if config.get("partition_by").granularity != config.get('batch_size') %}
```

Case-sensitive `!=` against un-lowercased values. `granularity`'s `day` default
is applied during parsing, so an omitted key is `None` here and fails the check —
`granularity` is effectively mandatory under `microbatch`.

Batch orchestration (`event_time`, `begin`, `lookback`, splitting, retry) is
dbt-core. No occurrences in dbt-adapters at the pinned commit; the only
`batch_size` reference outside this macro is an unrelated seed helper. Not
verified here.

## Schema change detection

`diff_columns` compares **names only** — source comment: "does not perform a data
type check". Type changes come from `diff_column_data_types`, which flags only
when `expanded_data_type` differs **and** `not sc.can_expand_to(other_column=tc)`.
Widening is therefore silent, emitting no `alter`.

`bigquery__check_for_schema_changes` sets `schema_changed` from
`source_not_in_target`, `target_not_in_source`, or `new_target_types`.

BigQuery override adds `adapter.sync_struct_columns(...)` before DDL, given the
chance to rewrite add/remove/retype lists — nested `STRUCT` field changes are
parent-level type changes, not top-level column adds. `source_relation` is
threaded through `changes_dict` for this hook, then `pop`ped.

`sync_all_columns` issues `alter_relation_add_remove_columns` plus per-column
`alter_column_type`. Drops target columns absent from source.

## Config aliasing

```jinja
config.get('predicates', default=none) or config.get('incremental_predicates', default=none)
```

`predicates` wins. `get_merge_sql` additionally honours a `predicates` kwarg via
`kwargs.get('predicates', incremental_predicates)` for back-compat.

---

Previous: [2. Generated SQL](02-generated-sql.md) ·
Next: [4. Quick reference](04-quick-reference.md)
