# E1 · One statement per model

> **Part E — Statement-level translation** · Sourcing: `SRC`
> **The question:** why can't I just paste my script into a model?

This is the constraint every other page in this track is downstream of. Get it
early and most of the rest stops being surprising.

## The rule

**A script is a sequence of statements. A model is one `select`.**

Your script says *do this, then this, then this*. A dbt model says *here is what
this table should contain*. The first is a procedure; the second is a definition.

You don't write `CREATE`, `INSERT`, `MERGE`, or `DELETE` in a model. You write
the `select`, and the materialization writes the DDL and DML around it.

## Where the constraint actually lives

It isn't a style rule. Look at the BigQuery incremental materialization
(`incremental.sql`) — your model's compiled code enters as a single value and is
consumed exactly once:

```jinja
{% set build_sql = bq_generate_incremental_build_sql(
    strategy, tmp_relation, target_relation, compiled_code, unique_key, ...
) %}

{%- call statement('main') -%}
  {{ build_sql }}
{% endcall %}
```

`compiled_code` is your model. It gets embedded — as a subquery in a `MERGE`, or
as the body of a `CREATE TABLE AS` — and one `statement('main')` runs the result.
There is no loop over your statements, because dbt never expected more than one.

Paste a second statement into a model and you aren't adding a step to a pipeline.
You're injecting a semicolon into the middle of somebody else's `MERGE`.

## The nuance that trips people up

**dbt does emit multi-statement scripts.** Dynamic `insert_overwrite` generates
this:

```sql
declare dbt_partitions_for_replacement array<date>;
create or replace table <tmp> as ( <your select> );
set (dbt_partitions_for_replacement) = ( select as struct array_agg(...) from <tmp> );
merge into <target> ... ;
drop table if exists <tmp>;
```

Five statements. So the constraint is not "BigQuery runs one statement" — it
plainly doesn't.

The constraint is on **who writes them**. dbt generates the procedure; you supply
the definition it wraps. Your script had both jobs tangled together, and
conversion is largely the work of separating them.

## What follows from it

Nearly every question that comes up during a conversion is this rule wearing a
different hat:

| Your script does | Why it doesn't fit | Where it goes |
| --- | --- | --- |
| Several statements in order | One `select` per model | [C1](../C-structural/C1-multi-statement-to-ctes.md) CTEs, [C2](../C-structural/C2-ephemeral-models.md) ephemeral, or [C3](../C-structural/C3-separate-models.md) separate models |
| Creates a temp table midway | You don't manage storage | A CTE, or its own model |
| Runs A then B because B needs A | Order isn't yours to state | `ref()` — [E2](../E-translation/E2-ordering-by-ref.md) |
| `DECLARE`s a variable | No statement to hold it | vars or macros — [C5](../C-structural/C5-declare-set-variables.md) |
| Writes to two tables | One model, one relation | Two models — [C4](../C-structural/C4-fan-out.md) |
| Grants, exports, notifies | Not part of a definition | Hooks — [Part F](../README.md#hooks--complete) |
| Loops | Nothing to loop over | Usually set-based logic — [C7](../C-structural/C7-loops.md) |

## The three legitimate exits

When something genuinely doesn't fit into one `select`, there are exactly three
places for it, in order of preference:

1. **Another model.** The default, and right far more often than people expect.
   If a step produces a meaningful intermediate result, it wants a name.
2. **A hook.** For side effects that belong to the model but aren't part of its
   definition — grants, table options, audit rows. Genuinely easy to abuse;
   [F17](../F-hooks/F17-when-a-hook-is-wrong.md) is about when not to.
3. **A `run-operation`.** For work that isn't a model at all — one-off
   maintenance, admin DDL. Not a scheduler; see
   [K4](../K-antipatterns/K4-run-operation-as-scheduler.md).

If your answer is "none of these", the honest conclusion is often that this
particular script shouldn't be a dbt model. That's [A7](../A-assess/A7-what-not-to-convert.md).

## Try it on your script

Before converting anything, answer one question: **which single statement
produces the table this script exists to build?**

That statement — usually the last `INSERT`, `MERGE`, or `CREATE TABLE AS` — is
your model. Everything above it is either setup (which becomes CTEs or upstream
models) or side effects (which become hooks, or leave). Everything below it is
side effects too.

If more than one statement produces a table you care about, you have more than
one model. That's not a complication; it's the answer.

---

Next in wave 1: [A3 · Classify by write pattern](../A-assess/A3-classify-by-write-pattern.md) ·
Back to [the backlog](../BACKLOG.md)
