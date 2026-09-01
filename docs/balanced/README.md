# The balanced track

The default reading. Assumes you know dbt and BigQuery — what a model is, what a
partition is, why a full scan costs money — but explains mechanisms rather than
just stating them.

## Read in order

1. [How the materialization runs](01-how-the-materialization-runs.md) — branch order, and when a temp table is or isn't created
2. [Choosing a strategy](02-choosing-a-strategy.md) — `merge` vs `insert_overwrite` vs `microbatch`
3. [The `merge` strategy](03-merge.md) — the generated `MERGE`, `unique_key` semantics, null handling
4. [The `insert_overwrite` strategy](04-insert-overwrite.md) — static vs dynamic, `copy_partitions`, the empty-partition trap
5. [The `microbatch` strategy](05-microbatch.md) — what's validated, what's delegated
6. [`partition_by` in detail](06-partition-config.md) — parsing and predicate rendering
7. [Schema changes](07-schema-changes.md) — `on_schema_change`, `STRUCT` synchronisation
8. [Gotchas](08-gotchas.md) — the behaviours that cost people a day

## The other tracks

Too dense? The [beginner track](../beginner/README.md) covers the same ground
from zero, at length.

Too slow? The [expert track](../expert/README.md) is the same knowledge as a
scannable reference.

---

Back to [the repository README](../../README.md)
