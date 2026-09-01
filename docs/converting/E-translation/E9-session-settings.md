# E9 · Session settings that spanned statements

> **Part E — Statement-level translation** · Sourcing: `SRC`
> **The question:** my script sets something at the top that later statements rely on. Where does that go?

A pre-hook, usually — it's one of the few things pre-hooks are genuinely for. The
catch is scope: dbt's statements aren't guaranteed to share a session the way a
script's do.

## The pattern

```sql
SET @@dataset_project_id = 'analytics-prod';
SET @@maximum_bytes_billed = 100000000000;

CREATE OR REPLACE TABLE daily_events AS SELECT ... ;
```

The `SET` applies to the session; subsequent statements inherit it.

## The conversion

```sql
{{ config(
    materialized='table',
    pre_hook="set @@maximum_bytes_billed = 100000000000"
) }}
```

Legitimate — see [F8](../F-hooks/F8-pre-hook-patterns.md). The pre-hook runs
before the main statement in the same connection, so the setting applies.

## The scope caveat

A script is one session. dbt runs each model on a connection from a pool, with
threads running models in parallel. A session setting applied in one model's
pre-hook is not guaranteed to affect another model.

So: **per-model settings work; project-wide settings via one model's hook do
not.** If every model needs the setting, put it in `on-run-start` — though even
that is per-connection, so the reliable place for a project-wide setting is the
profile or the query itself.

Don't set something in model A's pre-hook and depend on it in model B.

## What most of these become instead

Many session settings have better homes:

| Script sets | Better |
| --- | --- |
| `@@dataset_project_id` / `@@dataset_id` | The `database`/`schema` config, or the profile |
| `@@maximum_bytes_billed` | Profile-level, or a pre-hook for one expensive model — [E12](E12-cost-controls.md) |
| Query labels | The `labels` config, or a post-hook — [D7](../D-data-movement/D7-table-options.md) |
| Time zone | Explicit in the SQL — [H10](../H-verification/H10-reconciling-timestamps.md) |
| Legacy SQL | Nothing. dbt uses standard SQL |

The first row is the common one. A script setting a default project so later
statements can use bare table names is doing manually what `ref()` and `source()`
do properly — [E4](E4-cross-project-references.md). Don't port it; fix the
references.

## Values that spanned statements

If the "setting" is really a value the script computed once and reused, that's
[C5](../C-structural/C5-declare-set-variables.md), not a session setting.

The distinction: a session setting configures the *engine*; a `DECLARE`d value is
*data*. Engine settings go in hooks or the profile. Data goes in the model, as a
subquery or a Jinja variable.

## Timezone deserves care

```sql
SET @@time_zone = 'Europe/London';
```

If the script set this, every `CURRENT_DATE()`, `DATE()` extraction and
`DATETIME` conversion after it behaved differently from the default.

Don't reproduce it as a hook. **Make the timezone explicit in the SQL**:

```sql
date(occurred_at, 'Europe/London') as event_date
```

Explicit and local to where it matters, instead of an invisible setting that
changes the meaning of everything below it. And it removes a whole class of the
differences in [H10](../H-verification/H10-reconciling-timestamps.md).

Under `insert_overwrite` this isn't cosmetic — a timezone difference can move rows
across a partition boundary and overwrite the wrong day.

---

Previous: [D14 · Audit and metadata table writes](../D-data-movement/D14-audit-writes.md) ·
Next: [E10 · System variables](E10-system-variables.md)
