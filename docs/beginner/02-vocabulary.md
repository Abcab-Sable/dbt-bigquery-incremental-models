# 2. The words people will use at you

Every term you need, defined plainly, in the order it becomes relevant. Come back
to this page whenever something later assumes a word you don't have.

## The dbt words

**Model**
A single `.sql` file containing one `select` statement. That's genuinely all it
is. The file `clean_clicks.sql` is a model, and it produces a table or view
called `clean_clicks`.

**Materialization**
The *strategy dbt uses to turn your select statement into something that exists
in the database*. You set it with `materialized=...`. The main options:

| Materialization | What dbt does |
| --- | --- |
| `view` | Saves your query as a view. Nothing is stored; it re-runs each time someone reads it. |
| `table` | Runs your query and saves the results as a real table. Rebuilt from scratch each run. |
| `incremental` | Runs your query, then **adds the results into an existing table** instead of replacing it. |

This whole documentation set is about that third one.

**Relation**
dbt's generic word for "a table or a view". When you see `relation` in an error
message, read it as "the thing in the database this model produces".

**Run**
One execution of `dbt run`. Usually scheduled — every night, every hour.

**Full refresh**
Telling dbt to ignore the incremental machinery and rebuild the table completely,
with `dbt run --full-refresh`. Your escape hatch when an incremental model has
drifted from reality.

**Config**
Settings on a model, written in a block at the top of the file:

```sql
{{ config(
    materialized='incremental',
    unique_key='order_id'
) }}

select ...
```

Everything in this documentation is ultimately about which configs to set.

**Macro**
A reusable chunk of templating logic that dbt expands into SQL before running it.
You mostly don't write these — but dbt's *own* macros are what generate the SQL
for incremental models, and reading them is where all the facts in this
documentation come from.

**Compiled SQL**
Your model after dbt has expanded all the templating. The actual text sent to
BigQuery. You can see it in `target/compiled/` after a run, and looking at it is
the single most useful debugging habit you can build.

## The BigQuery words

**Dataset**
BigQuery's word for a folder of tables. Roughly what other databases call a
schema.

**Partition**
A table split into chunks, usually by date. A table partitioned on `event_date`
physically stores each day's rows separately.

This matters enormously, because BigQuery can read **one partition without
touching the others**. Filter to a single day, pay for a single day. It gets
[a whole page](04-partitioning-explained.md).

**Clustering**
Sorting the data inside each partition by some column, so filters on that column
read less. A secondary optimisation; partitioning is the one that changes how
incremental models behave.

**Pruning** (or *partition elimination*)
BigQuery noticing that your query only needs some partitions and skipping the
rest. This is the entire source of the cost saving. When people say a query
"doesn't prune", they mean BigQuery couldn't work out which partitions to skip,
so it read everything and you paid for everything.

**`MERGE`**
A SQL statement that combines insert, update, and delete in one operation. You
give it a source of rows, a target table, and a rule for matching them up:

```sql
merge into target_table as T
    using (select ... from new_data) as S
    on T.id = S.id

when matched then update set T.status = S.status
when not matched then insert (id, status) values (S.id, S.status)
```

Read as: for each row in the source, look for a matching row in the target; if
you find one, update it; if you don't, insert it.

**Nearly everything dbt does for incremental models on BigQuery is a `MERGE`.**
Different strategies just build different `MERGE` statements. Once you see that,
a lot of confusing behaviour becomes obvious.

**Temp table** (temporary table)
A throwaway table dbt sometimes creates to hold your model's output before
merging it into the real table. Dropped afterwards. Whether dbt creates one turns
out to matter a lot — see [page 5](05-the-three-strategies.md).

## The incremental words

**Strategy** (`incremental_strategy`)
*How* dbt combines new rows with existing ones. On BigQuery there are exactly
three valid values: `merge`, `insert_overwrite`, and `microbatch`. Anything else
is an error. If you don't set it, you get `merge`.

**`unique_key`**
The column (or columns) that identify a row as "the same row". If `order_id` is
your unique key, then an incoming row with `order_id = 5` will update the
existing row with `order_id = 5` rather than adding a second one.

**Leave it out and dbt cannot tell rows apart, so it inserts everything.** That
is a real and common source of duplicates.

**`partition_by`**
The config telling dbt to partition the table, and on which column. Required for
two of the three strategies.

**`is_incremental()`**
A function you use inside your model to add filtering that only applies on
incremental runs. Covered properly on [the next page](03-your-first-incremental-model.md).

**`on_schema_change`**
What dbt should do when your model starts producing different columns than the
table already has. Defaults to `ignore`, which does exactly what it says and
surprises people constantly.

**Late-arriving data**
Rows that show up after you've already processed the time period they belong to.
The reason naive "only take rows newer than my newest row" filters lose data.

**Backfill**
Re-processing a historical period, usually because you fixed a bug or added a
column. Often the thing that makes people discover their incremental logic was
wrong.

**Idempotent**
A run you can repeat without changing the outcome. Running it twice gives the
same table as running it once. This is a property you want and don't get for
free — an append-only model run twice gives you every row twice.

## A quick self-test

If these sentences make sense, you're ready for the next page:

- "It's materialized incremental with a unique key, so it merges on that key."
- "It's partitioned by day, so filtering to one date only reads one partition."
- "The compiled SQL is a MERGE that doesn't prune, which is why it costs so much."

If they don't yet, that's expected — reread the definitions above once. Every one
of those sentences is unpacked in the pages that follow.

---

Previous: [1. What problem are we even solving?](01-what-problem-are-we-solving.md) ·
Next: [3. Your first incremental model](03-your-first-incremental-model.md)
