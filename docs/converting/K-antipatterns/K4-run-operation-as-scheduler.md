# K4 · `run-operation` used as a scheduler

> **Part K — Anti-patterns** · Sourcing: `CRAFT`
> **The question:** can I just wrap the script in a macro and call `dbt run-operation`?

You can. It produces a script that lives in a dbt project and gets none of dbt's
benefits — which is strictly worse than leaving it as a script.

## The shape

```sql
{% macro run_daily_pipeline() %}
  {% do run_query("create or replace table analytics.stg_events as select ...") %}
  {% do run_query("create or replace table analytics.enriched as select ...") %}
  {% do run_query("create or replace table analytics.daily_summary as select ...") %}
{% endmacro %}
```

```bash
dbt run-operation run_daily_pipeline
```

It works. It is the original script, transliterated into Jinja.

## What it costs

**No DAG.** The three tables have no lineage, no `ref()`, no ordering except the
line order you wrote — which is what conversion was supposed to remove.

**No models.** Nothing to `--select`, test, document, or see in the docs site.

**No materialization logic.** All the incremental machinery documented here is
unavailable; you're writing `CREATE OR REPLACE` by hand.

**Worse debugging than the script.** SQL built as strings inside Jinja, so
`target/compiled/` shows you nothing useful.

**Harder to read.** `run_query` inside `{% do %}` obscures perfectly ordinary SQL.

You have added a templating layer and removed every feature that justified it.

## What `run-operation` is actually for

Genuine one-off or administrative work that isn't a model:

```bash
dbt run-operation drop_old_partitions --args '{dataset: analytics, days: 90}'
dbt run-operation grant_reader --args '{principal: "group:new-team@acme.com"}'
dbt run-operation create_udfs
```

Characteristics: not producing a modelled table, not on the critical path of a
build, invoked deliberately rather than on a schedule.

`create_udfs` is a good example — [C11](../C-structural/C11-temp-functions.md).
So is a maintenance task run from a deploy pipeline.

## The tell

If the operation:

- produces tables other things read
- runs on a schedule
- contains statements in a required order
- would benefit from lineage

…it's a set of models. Convert it properly —
[C12](../C-structural/C12-nested-procedures.md) covers exactly this shape when
the script is a procedure call tree.

## Why people reach for it

Usually one of:

**The logic is procedural** and doesn't fit one `select`. That's
[C6](../C-structural/C6-if-branching.md) and
[C7](../C-structural/C7-loops.md) — and the honest answer is often that it stays
outside dbt entirely, which is fine ([A7](../A-assess/A7-what-not-to-convert.md)).

**It's a quick win under deadline pressure.** Understandable, and it becomes
permanent. If you do it, put a ticket and an expiry in a comment.

**It writes to tables dbt doesn't own.** Then it isn't a model — but it also
probably isn't dbt's job. Your orchestrator can run SQL.

## Leaving it as a script is better

If the work genuinely can't be modelled, leaving it as a script and declaring its
output as a `source` gives you lineage for downstream models with none of the
pretence — [I2](../I-migration/I2-strangler-pattern.md).

A script that admits it's a script is more honest, and easier to maintain, than a
macro pretending to be a conversion.

---

Previous: [K3 · Incremental where a table belongs](K3-unnecessary-incremental.md) ·
Next: [K5 · Imperative structure in Jinja](K5-imperative-jinja.md)
