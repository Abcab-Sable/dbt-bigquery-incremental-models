# The expert track

Condensed reference. Assumes fluency in dbt, BigQuery pricing and partition
semantics, and Jinja macro dispatch. No motivation, no worked examples.

Four pages:

1. [Control flow and dispatch](01-control-flow.md) — branch order, macro chain, temp-relation conditions
2. [Generated SQL](02-generated-sql.md) — emitted statements per path
3. [Semantics and edge cases](03-semantics.md) — null handling, rendering, version delta
4. [Quick reference](04-quick-reference.md) — config matrix, error catalogue, source map

Fuller treatment in the [balanced track](../balanced/01-how-the-materialization-runs.md).
Ground-up explanation in the [beginner track](../beginner/README.md).

## Orientation in one screen

BigQuery overrides the default incremental materialization entirely. No
intermediate relation, no `rename_relation` swap. Three strategies, of which
`microbatch` delegates verbatim to the `insert_overwrite` builder — identical
emitted SQL, differing only in compile-time validation and dbt-core's batch
orchestration, which is now pinned too.

All three strategies emit a `MERGE`. `insert_overwrite` is cheaper than `merge`
not because it avoids `MERGE` but because its predicate bounds the target side to
a prunable partition set.

The three highest-value facts, if you read nothing else:

- **`on_schema_change != 'ignore'` forces temp-relation creation**, converting an
  inlined single-statement `merge` into `CREATE` + `MERGE`. It is a plan-shape
  config disguised as a schema config.
- **Dynamic `insert_overwrite` derives its replacement set from produced rows.**
  Empty output ⇒ empty `array_agg` ⇒ no delete. Deletions never propagate.
- **Composite `unique_key` bypasses `get_merge_unique_key_match`** and emits bare
  `=` per column. `enable_truthy_nulls_equals_macro` does not reach it.

Provenance, including the one released-vs-`main` behavioural delta, is in
[the repository README](../../README.md).
