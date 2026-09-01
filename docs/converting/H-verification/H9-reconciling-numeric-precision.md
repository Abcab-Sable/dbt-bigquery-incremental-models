# H9 · Reconciling: float and `NUMERIC` precision

> **Part H — Proving correctness** · Sourcing: `CRAFT`
> **The question:** the totals differ by 0.0000001. Is that a bug?

Almost certainly not — but you have to be able to say why, and you have to stop
comparing floats for exact equality.

## Why it happens

`FLOAT64` addition is not associative. `(a + b) + c` can differ from
`a + (b + c)` in the last bits. BigQuery parallelises aggregation and doesn't
guarantee summation order, so **the same query over the same data can produce
slightly different sums between runs** — never mind between two implementations.

So a tiny difference in a `SUM(FLOAT64)` is expected. Exact equality was never a
reasonable bar.

## Compare with a tolerance

```sql
select
    countif(abs(old.total - new.total) > 0.01)                    as material_diffs,
    countif(abs(old.total - new.total) between 1e-9 and 0.01)     as rounding_diffs,
    max(abs(old.total - new.total))                               as worst_diff
from analytics.revenue old
join analytics_dbt.revenue new using (day);
```

`material_diffs` zero with a non-zero `rounding_diffs` is a pass. Set the
tolerance from what the number means — for currency, anything below half a penny
is noise.

Relative tolerance is better for values spanning orders of magnitude:

```sql
countif(abs(old.total - new.total) / nullif(abs(old.total), 0) > 1e-9) as relative_diffs
```

## This breaks hashing

[H3](H3-checksum-parity.md)'s fingerprint approach compares serialised values, so
a last-bit difference produces a completely different hash. A float column will
make a hash comparison fail while the data is fine.

**If the table has floats, don't hash it.** Either exclude the float columns from
the hash and compare them separately with a tolerance, or use column-level
comparison throughout.

## `NUMERIC` doesn't have this problem

`NUMERIC` and `BIGNUMERIC` are exact decimal types. Addition is associative,
results are deterministic, and exact comparison is meaningful.

If the values are money, they should be `NUMERIC`. If your script used `FLOAT64`
for currency, that's a pre-existing bug — note it in
[A6](../A-assess/A6-compensating-hacks.md), and fix it as a separate change, not
during conversion ([K11](../BACKLOG.md#part-k--anti-patterns)).

Watch for the type changing during conversion. A `CREATE TABLE` declaring
`NUMERIC(10,2)` becomes whatever your `select` infers unless you cast —
[B2](../B-write-patterns/B2-create-if-not-exists.md):

```sql
select cast(sum(amount) as numeric) as total
```

## Division and rounding

Two more sources of legitimate difference:

**Integer division.** If the script divided integers somewhere and your model
casts first, results differ by the fractional part. Check for `/` on integer
columns.

**Rounding order.** `ROUND(SUM(x), 2)` and `SUM(ROUND(x, 2))` differ, sometimes
materially over many rows. Preserve the original order of operations exactly; this
is a case where "tidying" the SQL changes the answer.

## Record the tolerance

In your [H1](H1-what-correct-means.md) scope:

```
Numeric comparison: revenue.total compared with absolute tolerance 0.01
                    (FLOAT64, summation order not guaranteed).
                    order_count compared exactly (INT64).
```

An unstated tolerance turns a proof into a claim. And a tolerance chosen after
seeing the diff is a tolerance chosen to pass.

---

Previous: [H8 · Reconciling: nulls](H8-reconciling-nulls.md) ·
Next: [H10 · Reconciling: timestamps and timezones](H10-reconciling-timestamps.md)
