# Statement-pattern cheatsheet

One page. Find the shape of the SQL you have, read across, follow the link if the
row surprises you.

Everything here is BigQuery-specific and matches `dbt-bigquery` 1.12.0 /
`dbt-core` 1.12.3. Three incremental strategies exist — `merge`,
`insert_overwrite`, `microbatch`. **There is no `delete+insert` and no `append`
strategy on BigQuery**; rows below that look like they need one are doing it
another way. Nothing expresses an arbitrary `DELETE` either — see §4.

---

## 1. Pick the strategy

| Your script | Materialization | Why |
| --- | --- | --- |
| `CREATE OR REPLACE TABLE` on the whole thing | `table` | No incrementality to preserve — [B1](B-write-patterns/B1-create-or-replace-to-table.md), and don't invent it: [K3](K-antipatterns/K3-unnecessary-incremental.md) |
| `TRUNCATE` + `INSERT` | `table`, or `insert_overwrite` | `table` if the reload is unbounded; `insert_overwrite` if it only reloads recent data — [B15](B-write-patterns/B15-truncate-insert.md) |
| `CREATE VIEW` | `view` | [B3](B-write-patterns/B3-create-view.md) |
| `MERGE` keyed on an id | `incremental`, `merge` | dbt generates the same statement — [B8](B-write-patterns/B8-merge-on-clause-to-unique-key.md) |
| `DELETE` a date range + `INSERT` it back | `incremental`, `insert_overwrite` | One atomic `MERGE` instead of two statements — [B13](B-write-patterns/B13-delete-insert-to-insert-overwrite.md) |
| `DELETE` on anything else + `INSERT` | depends on the `WHERE` | Eight shapes, and only some are `insert_overwrite` — **§4** |
| `INSERT` filtered on a watermark | `incremental`, `merge`, no `unique_key` | Append; watermark becomes `is_incremental()` — [B6](B-write-patterns/B6-watermark-filter.md) |
| `INSERT`, watermark held in a state table | `incremental` | The state table becomes `{{ this }}` — an incremental model already has somewhere to keep state — [B7](B-write-patterns/B7-external-watermark.md) |
| Unfiltered `INSERT ... SELECT` | `incremental`, `merge`, no `unique_key` | **First find out why it isn't already duplicating** — usually a transient source. Converting exposes the bug — [B5](B-write-patterns/B5-unfiltered-insert.md) |
| Per-day loop over partitions | `microbatch` | [B14](B-write-patterns/B14-when-the-range-can-empty.md), [C7](C-structural/C7-loops.md) |
| Dedup rewrite of the whole table | `table` or dedup-in-model | [B16](B-write-patterns/B16-deduplication.md) |

`merge` costs scale with **target size**; `insert_overwrite` and `microbatch`
with **partitions touched**. `insert_overwrite` and `microbatch` both *require*
`partition_by` and both *ignore* `unique_key`.

---

## 2. `MERGE`, clause by clause

```sql
MERGE INTO analytics.orders AS T
USING ( <subquery> ) AS S
ON T.order_id = S.order_id AND T.order_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
WHEN MATCHED AND S.updated_at > T.updated_at THEN UPDATE SET ...
WHEN NOT MATCHED THEN INSERT (...) VALUES (...)
WHEN NOT MATCHED BY SOURCE THEN DELETE;
```

| Clause | Where it goes | Note |
| --- | --- | --- |
| `MERGE INTO <table>` | the model's name | — |
| `USING ( ... )` | **the model body** | This is your `select`. [E1](E-translation/E1-one-statement-per-model.md) |
| `ON T.k = S.k` | `unique_key='k'` | [B8](B-write-patterns/B8-merge-on-clause-to-unique-key.md) |
| `ON` … extra key columns | `unique_key=['a','b']` | **Not null-safe** — see §6 |
| `ON` … non-key predicates | `incremental_predicates=[...]` | Use `DBT_INTERNAL_DEST` / `DBT_INTERNAL_SOURCE`, not `T`/`S` — [B12](B-write-patterns/B12-extra-predicates.md) |
| `WHEN MATCHED THEN UPDATE SET` | generated, all columns | Narrow with `merge_update_columns` **xor** `merge_exclude_columns` — [B9](B-write-patterns/B9-when-matched-update.md) |
| `WHEN MATCHED AND <cond>` | **the model body** | No config for it. Usually becomes dedup/qualify — [B11](B-write-patterns/B11-conditional-when-matched.md) |
| `WHEN MATCHED AND ... THEN DELETE` | model body (filter the row out) | [B11](B-write-patterns/B11-conditional-when-matched.md) |
| `WHEN NOT MATCHED THEN INSERT` | generated | — |
| `WHEN NOT MATCHED BY SOURCE THEN DELETE` | **no equivalent** | `merge` emits two branches only. Options in [B10](B-write-patterns/B10-not-matched-by-source.md) |
| the `WHERE ... > (SELECT MAX ...)` inside `USING` | `{% if is_incremental() %}` block | [B6](B-write-patterns/B6-watermark-filter.md) |

---

## 3. Combinations, in one script

| Script shape | Do this |
| --- | --- |
| `DELETE` range, then `INSERT` range | `insert_overwrite` with `partitions=[...]` — never a `pre_hook` delete ([F9](F-hooks/F9-pre-hook-deletes.md)) |
| `DELETE` on a non-partition predicate, then `INSERT` | Classify the `WHERE` first — **§4** |
| `MERGE`, then a separate `DELETE` | Two concerns: an upsert and a purge. Split them — **§4** |
| `UPDATE` then `INSERT` (upsert by hand) | One `merge` with `unique_key`. The `UPDATE`'s `SET` list ⇒ `merge_update_columns` |
| `MERGE` then a second `UPDATE` to patch columns | Fold the patch into the model body; a second write to `{{ this }}` is [K2](K-antipatterns/K2-hooks-as-escape-hatch.md) |
| `MERGE` into two tables | Two models — [C4](C-structural/C4-fan-out.md) |
| `DECLARE` / `SET`, then statements using the variable | Jinja `{% set %}` or `var()`; not a runtime variable — [C5](C-structural/C5-declare-set-variables.md), [E11](E-translation/E11-query-parameters.md) |
| `IF` branching around which write happens | Usually two models plus a selector, not Jinja `{% if %}` — [C6](C-structural/C6-if-branching.md), [K5](K-antipatterns/K5-imperative-jinja.md) |
| `LOOP` over dates | `microbatch`, or `--full-refresh` over a range — [C7](C-structural/C7-loops.md), [G6](G-scheduling/G6-backfill-microbatch.md) |
| Temp tables between statements | CTEs ([C1](C-structural/C1-multi-statement-to-ctes.md)) → ephemeral ([C2](C-structural/C2-ephemeral-models.md)) → separate models ([C3](C-structural/C3-separate-models.md)), in that order of preference |
| `BEGIN TRANSACTION` / `COMMIT` | Nothing. dbt has no multi-statement transaction — [C9](C-structural/C9-transactions.md) |
| `EXECUTE IMMEDIATE` building SQL | Jinja at compile time, or admit it's orchestration — [C10](C-structural/C10-dynamic-sql.md) |
| `ASSERT` before the write | dbt test on the upstream model — [D12](D-data-movement/D12-assert-gates.md) |
| `ALTER TABLE ADD COLUMN` | `on_schema_change` — but read §6, it is also a cost setting — [D8](D-data-movement/D8-add-column-migrations.md) |
| `GRANT` after the write | `grants` config, not a `post_hook` — [F11](F-hooks/F11-grants-vs-post-hook.md) |
| `EXPORT DATA` at the end | Orchestration, not a hook — [D2](D-data-movement/D2-export-data.md) |

---

## 4. `DELETE` when the predicate isn't a partition range

`insert_overwrite` is the answer only when the `DELETE`'s `WHERE` lines up with
partition boundaries. Often it doesn't. The reframe that makes every other case
tractable:

> A dbt model declares **what the table should contain**, never **what to
> remove**. So every `DELETE` has to be re-expressed as *which rows survive* —
> and the scope you can say that over is either the whole table (`table`) or a
> set of partitions (`insert_overwrite`). There is no third scope.

Classify by what the `WHERE` touches:

| `DELETE ... WHERE` | It's really | Convert as |
| --- | --- | --- |
| the partition column, any expression | partition replacement | `insert_overwrite`. Enumerable ⇒ static `partitions=[...]` — [B13](B-write-patterns/B13-delete-insert-to-insert-overwrite.md) |
| the same keys the `INSERT` supplies | an upsert written longhand | `merge` + `unique_key` — [B8](B-write-patterns/B8-merge-on-clause-to-unique-key.md) |
| a non-partition column, rows confined to partitions you can compute | partition replacement, widened | `insert_overwrite` over the superset — **see below** |
| an attribute with no partition relationship (`status = 'cancelled'`) | a redefinition of the whole table | `table`, or soft-delete — [B10](B-write-patterns/B10-not-matched-by-source.md) |
| `IN (SELECT ... FROM other_table)` | an anti-join | `ref()` the other table, `where not exists` in the body — **see below** |
| a range **wider** than the `INSERT` reloads | two concerns tangled | Split: a reload *and* a retention purge |
| `date < <cutoff>`, nothing reinserted | retention, not a model | Partition expiration or a scheduled op — [K2](K-antipatterns/K2-hooks-as-escape-hatch.md), [K4](K-antipatterns/K4-run-operation-as-scheduler.md) |
| nothing (`DELETE FROM t;`) | a replace | `table` — [B1](B-write-patterns/B1-create-or-replace-to-table.md) |

### Widening to a partition superset

The useful move when the delete predicate isn't the partition column, but the
affected rows are reachable from it.

```sql
-- deletes by a non-partition column
DELETE FROM analytics.orders WHERE customer_id IN (SELECT customer_id FROM raw.churned);
INSERT INTO analytics.orders SELECT ... FROM raw.orders WHERE ...;
```

If those customers' orders all fall inside a known partition range, overwrite
**those partitions in full**:

```sql
{{ config(
    materialized='incremental',
    incremental_strategy='insert_overwrite',
    partition_by={'field': 'order_date', 'data_type': 'date', 'granularity': 'day'},
    partitions=['date_sub(current_date(), interval 30 day)', '...']
) }}

select * from {{ source('raw', 'orders') }}
where order_date >= date_sub(current_date(), interval 30 day)
  and customer_id not in (select customer_id from {{ ref('churned') }})
```

**The trap, and it is the one that catches everyone:** once you overwrite a
partition, the model must emit *every row that should remain in it* — not just
the rows that changed. `insert_overwrite` replaces partitions wholesale. Emit
only the delta and you have silently deleted every untouched row in those
partitions. The run succeeds.

This is only viable while the superset stays small. If a row from three years ago
can be deleted today, you'd overwrite three years of partitions every run, and
`table` is both cheaper and correct by construction.

### Deletes driven by another table

```sql
DELETE FROM analytics.orders WHERE order_id IN (SELECT order_id FROM ops.deletion_log);
```

The deletion log becomes a first-class dependency, and the delete becomes an
anti-join in the model body:

```sql
select o.* from {{ source('raw', 'orders') }} o
where not exists (
    select 1 from {{ ref('deletion_log') }} d where d.order_id = o.order_id
)
```

Two things to get right. The log has to be **cumulative** — if it's purged after
each run, the anti-join stops excluding rows the moment the log empties, and
previously-deleted rows reappear. And this only removes rows on a scope you
rewrite, so pair it with `table`, or with `insert_overwrite` over partitions wide
enough to cover the log's reach.

### When none of these fit

You have an unbounded, arbitrary delete. Three honest answers, in order:

1. **`materialized='table'`.** Correct by construction, no drift. Run the cost
   numbers before dismissing it — [A7](A-assess/A7-what-not-to-convert.md).
2. **Soft deletes.** Upstream marks rows deleted instead of removing them;
   deletion becomes an update, which `merge` handles natively, and you keep the
   audit trail. Needs a conversation with whoever owns the source.
3. **Keep it out of dbt.** A genuine purge — GDPR erasure, retention — is an
   operation on a table, not a model that defines one.

What isn't on the list: a `pre_hook` or `post_hook` delete. Not atomic, invisible
to the materialization, and on failure it leaves deleted data with no
replacement — [F9](F-hooks/F9-pre-hook-deletes.md),
[F17](F-hooks/F17-when-a-hook-is-wrong.md).

---

## 5. Config surface

| Config | Default | The thing that bites |
| --- | --- | --- |
| `unique_key` | none | Absent ⇒ `on FALSE` ⇒ **append-only** |
| `unique_key` as a list | — | Bare `=`, never null-safe |
| `incremental_predicates` | none | ANDed into `ON`. A target-side bound ⇒ rows outside it **insert instead of update** |
| `merge_update_columns` | all | Mutually exclusive with `merge_exclude_columns`; both ⇒ compile error |
| `on_schema_change` | `ignore` | Anything but `ignore` **forces a temp table** — one `MERGE` becomes two statements |
| `partition_by` | none | String values are lowercased at parse time |
| `partitions` | none | Non-empty ⇒ static `insert_overwrite`; interpolated **verbatim**, so granularity must match |
| `copy_partitions` | `false` | `insert_overwrite`/`microbatch` only; one API call per partition |
| `require_partition_filter` | `false` | dbt satisfies it with a tautology — **no pruning benefit** |
| `batch_size` | none | Must equal `partition_by.granularity`, case-sensitively |

Full table: [expert quick reference](../expert/04-quick-reference.md#config--behaviour).

---

## 6. Edge cases

Ordered by how often they cost a day.

| Symptom | Cause | Fix |
| --- | --- | --- |
| Rows keep appearing, updates never apply | No `unique_key` on `merge` ⇒ `on FALSE` ⇒ append-only | Set `unique_key`, or you wanted `insert_overwrite` — [B8](B-write-patterns/B8-merge-on-clause-to-unique-key.md) |
| A partition that should be empty keeps its old rows | Dynamic `insert_overwrite` derives the replacement set from rows the model produced. No rows ⇒ no delete. **The run succeeds.** | Static `partitions=[...]`, which deletes the listed partitions regardless — [B14](B-write-patterns/B14-when-the-range-can-empty.md) |
| Duplicates on a composite key | List `unique_key` uses bare `=`; a `NULL` component never matches. `enable_truthy_nulls_equals_macro` does **not** help — the list branch never reaches the null-safe macro | `coalesce` into a surrogate key — [expert](../expert/03-semantics.md#unique_key-three-branches-two-behaviours) |
| Duplicates on a single key, only for old rows | `incremental_predicates` window narrower than the data actually changes | Widen the bound, or accept and reconcile — [B12](B-write-patterns/B12-extra-predicates.md) |
| Upstream deletes never propagate | `merge` has no `when not matched by source` | `insert_overwrite` over a snapshot range, or a soft-delete flag — [B10](B-write-patterns/B10-not-matched-by-source.md) |
| New column silently missing | `on_schema_change='ignore'` takes columns from the **target** | Set `sync_all_columns` — and accept the temp table — [balanced](../balanced/07-schema-changes.md) |
| Model got 2× slower after adding `on_schema_change` | Non-`ignore` forces a temp relation | Expected. It's a cost/safety trade — [balanced](../balanced/01-how-the-materialization-runs.md#when-a-temp-table-is-created-and-when-it-isnt) |
| Rows with a `NULL` partition value accumulate | `array_agg(distinct ... IGNORE NULLS)` excludes them from the replacement set | Never let the partition column be null — [balanced](../balanced/04-insert-overwrite.md) |
| Rows vanished from partitions the change never touched | Overwrote a partition superset but emitted only the changed rows. `insert_overwrite` replaces partitions **wholesale** | Emit every surviving row in every partition you overwrite — §4 |
| Deleted rows came back a run later | Anti-join against a deletion log that gets purged between runs | The log must be cumulative — §4 |
| Deletes propagate for recent rows, never for old ones | Overwrite window narrower than how far back a delete can reach | Widen, or `table` — §4 |
| `pre_hook` deleted the data and the model then failed | Not atomic; two statements | `insert_overwrite` — [F9](F-hooks/F9-pre-hook-deletes.md) |
| `microbatch` hooks fired once, not per batch | `pre_hook` on the first batch, `post_hook` on the last | By design — [balanced](../balanced/05-microbatch.md) |
| Static `partitions` list replaced nothing | Literals compared against the **rendered** partition expression; wrong granularity ⇒ no match | Match the granularity exactly — [balanced](../balanced/06-partition-config.md) |
| Backfill rewrote partitions you didn't mean to | `--full-refresh` replaces the whole table | Partition-range backfill instead — [G7](G-scheduling/G7-backfill-partition-ranges.md) |
| Late data lands outside the incremental window | Watermark filter is tighter than arrival latency | Widen the lookback — [G8](G-scheduling/G8-late-arriving-data.md) |
| Everything green, numbers wrong | Green means "no error", not "correct" | [K12](K-antipatterns/K12-trusting-green-runs.md), then [H2](H-verification/H2-row-count-parity.md) |

---

## 7. No dbt equivalent — stop looking

| Construct | Reality |
| --- | --- |
| `WHEN NOT MATCHED BY SOURCE THEN DELETE` | Not expressible in `merge` — [B10](B-write-patterns/B10-not-matched-by-source.md) |
| Conditional `WHEN MATCHED AND ...` | Move it into the model body — [B11](B-write-patterns/B11-conditional-when-matched.md) |
| Multi-statement transactions | Don't exist — [C9](C-structural/C9-transactions.md) |
| `EXCEPTION WHEN ERROR` | Failure is the orchestrator's job — [C8](C-structural/C8-exception-handling.md) |
| Session settings | Per-statement only — [E9](E-translation/E9-session-settings.md) |
| Row-level security carried by the model | Applied to the relation, not generated — [D11](D-data-movement/D11-policy-tags-rls.md) |

---

## 8. Before you call it converted

1. It is idempotent — run it twice, get the same table — [E8](E-translation/E8-idempotency-proving.md).
2. Row counts match the old table, per partition — [H2](H-verification/H2-row-count-parity.md).
3. Columns match, not just counts — [H4](H-verification/H4-column-level-diffing.md).
4. Differences that *should* exist are written down — [H11](H-verification/H11-differences-that-should-exist.md).
5. A scheduled full-refresh reconciliation exists — [J3](J-operating/J3-scheduled-reconciliation.md).

Don't fix the script's bugs in the same change that converts it —
[K6](K-antipatterns/K6-porting-the-bug.md), [K11](K-antipatterns/K11-convert-and-optimise.md).
