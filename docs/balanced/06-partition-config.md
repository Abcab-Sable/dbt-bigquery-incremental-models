# 6. `partition_by` in detail

Source: `dbt-bigquery/src/dbt/adapters/bigquery/relation_configs/_partition.py`

`partition_by` is not just table DDL. The same config object generates the
predicates that decide which rows `insert_overwrite` deletes, so getting it wrong
produces wrong data, not just a slow query.

## The fields

```python
field: str
data_type: str = "date"
granularity: str = "day"
range: Optional[Dict[str, Any]] = None
time_ingestion_partitioning: bool = False
copy_partitions: bool = False
```

`PartitionConfig.parse` **lowercases every string value** before constructing the
object:

```python
{key: (value.lower() if isinstance(value, str) else value)
 for key, value in raw_partition_by.items()}
```

So `data_type: 'DATE'` and `data_type: 'date'` are equivalent — and your `field`
name is lowercased too. On BigQuery, column names are case-insensitive for
resolution, so this is normally invisible. It stops being invisible in the
`microbatch` granularity check, which compares against the **raw** dict rather
than the parsed object (see [page 5](05-microbatch.md#the-granularity-check)).

A malformed config raises one of two errors: `DbtValidationError` ("Could not
parse partition config") from a `ValidationError`, or a `CompilationError`
naming the expected shape if it isn't dict-like at all.

## How the partition expression is rendered

Two methods, and the distinction between them is the important part.

`render(alias)` produces the expression used for *matching*:

```python
if self.data_type_should_be_truncated():
    return f"{self.data_type}_trunc({column}, {self.granularity})"
else:
    return column
```

`data_type_should_be_truncated()` returns `False` in exactly two cases:

```python
return not (self.data_type == "int64"
            or (self.data_type == "date" and self.granularity == "day"))
```

So the raw column is used for `int64`, and for daily date partitioning. Every
other combination gets wrapped in `<type>_trunc(col, <granularity>)` —
`timestamp_trunc(ts, hour)`, `date_trunc(d, month)`, and so on.

`render_wrapped(alias)` builds on that and is what the strategies actually call.
It adds two normalisations.

### Time types get a cast

```python
if (self.data_type in ("date", "timestamp", "datetime")
        and not self.data_type_should_be_truncated()
        and not (self.time_ingestion_partitioning and self.data_type == "date")):
    return f"{self.data_type}({self.render(alias)})"
```

The condition is only satisfiable in the daily-date case (the other
non-truncated type is `int64`, which isn't in the tuple), so in practice this
yields `date(<column>)`. The `time_ingestion_partitioning` exclusion is there
because `_PARTITIONDATE` is already a `DATE` — the comment says exactly that.

### `int64` range partitions get boundary normalisation

```python
if (self.data_type == "int64" and self.range is not None
        and self.range["interval"] not in (1, "1")):
    return f"({column} - MOD({column} - {start}, {interval}))"
```

This maps any value to the start of its partition. Without it, `array_agg(distinct
...)` in the dynamic `insert_overwrite` path would collect every distinct *value*
rather than every distinct *partition* — the docstring notes this "prevents
generating excessively large arrays of distinct values". With `range: {start: 0,
end: 1000000, interval: 1000}` and a million distinct ids, you would otherwise
build a million-element array to describe a thousand partitions.

> **Version difference.** The `interval not in (1, "1")` guard is present on
> `main` at the pinned commit but **not in released 1.12.0**. On 1.12.0 the
> normalisation is applied whenever `range` is set, so `interval: 1` emits
> `(col - MOD(col - start, 1))` — arithmetically a no-op, but an expression
> rather than a bare column, which blocks partition elimination. On `main` the
> raw column is returned instead, "to preserve partition elimination".
>
> If you use `int64` range partitioning with `interval: 1` on 1.12.0, expect
> `insert_overwrite` to scan more than you think. This is the only behavioural
> difference found between the released package and the pinned commit.

## Where each rendering is used

| Call site | Expression | Purpose |
| --- | --- | --- |
| Static `insert_overwrite` | `render_wrapped(alias='dbt_internal_dest')` | `... in (<your literals>)` |
| Dynamic `insert_overwrite` | `render_wrapped(alias='DBT_INTERNAL_DEST')` | `... in unnest(dbt_partitions_for_replacement)` |
| Dynamic partition discovery | `render_wrapped()` | `select distinct ... from <tmp>` |
| Dynamic + `copy_partitions` | `render_wrapped()` | `select distinct ... from <tmp>` |
| `require_partition_filter` guard | raw field, or `time_partitioning_field()` under ingestion-time | the tautology predicate |
| `_dbt_max_partition` | `partition_by.field` (raw) | `select max(field) ...` |

The last two use the **raw field**, not a rendered expression. So
`_dbt_max_partition` on an hourly timestamp partition gives you the maximum
*timestamp*, not the start of the maximum hour.

Because both sides of the `in` comparison are rendered the same way, the
comparison is consistent. The value you must supply yourself is the literal list
in static mode — those are compared against the *rendered* destination
expression, so `partitions` values must be at partition granularity. For monthly
partitioning, `'2026-08-15'` will not match `date_trunc(dest, month)`; you need
`'2026-08-01'`.

## `data_type_for_partition`

Used to type the `declare dbt_partitions_for_replacement array<...>` statement:

```python
if not self.time_ingestion_partitioning:
    return self.data_type
return "date" if self.data_type == "date" else "timestamp"
```

With ingestion-time partitioning, the array collapses to `date` or `timestamp` —
matching `_PARTITIONDATE` / `_PARTITIONTIME`.

## Ingestion-time partitioning

Set `time_ingestion_partitioning: true` and dbt partitions on BigQuery's
pseudo-column instead of a real column. Source:
`incremental_strategy/time_ingestion_tables.sql`.

The model SQL is rewritten:

```sql
select TIMESTAMP(<field>) as _PARTITIONTIME, * EXCEPT(<field>) from ( <your sql> )
```

Your partition column is **removed from the output** and replaced by
`_PARTITIONTIME`. Downstream models cannot select it by its original name.

`time_partitioning_field()` returns `_PARTITIONDATE` for `data_type: date` and
`_PARTITIONTIME` otherwise — the docstring notes the mismatch "will fail
statements for type mismatch". But `insertable_time_partitioning_field()` always
returns `_PARTITIONTIME`, with the comment "Practically, only _PARTITIONTIME
works so far." Reads and writes therefore use different pseudo-columns under
`data_type: date`.

Also note the table must be created before data is inserted — from
`bq_create_table_as`: "ingestion time partitioned tables can't be created with
the transformed data". So this path always runs a `CREATE` then an `INSERT`,
never a `CREATE TABLE AS`.

And, as on [page 1](01-how-the-materialization-runs.md#python-models), Python
models reject it outright.

---

Previous: [5. The `microbatch` strategy](05-microbatch.md) ·
Next: [7. Schema changes](07-schema-changes.md)
