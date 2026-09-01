# I2 · The strangler pattern for a script suite

> **Part I — Migration strategy** · Sourcing: `CRAFT`
> **The question:** how do we migrate without a big-bang cutover?

Grow the dbt project around the scripts, replacing them one at a time, until
there's nothing left. Never a moment where everything moves at once.

## The shape

1. **Wrap.** Declare every script's output as a `source`. Nothing changes
   operationally; you gain lineage and freshness checks.
2. **Replace one.** Convert a script to a model. Delete its source declaration;
   swap `source()` for `ref()` in its consumers.
3. **Repeat.** Each conversion shrinks the script suite by one.
4. **Remove the scaffolding.** When no sources remain pointing at script outputs,
   the migration is done.

Step 1 is the one people skip, and it's the cheapest value in the entire
migration.

## Step 1 in practice

```yaml
sources:
  - name: legacy
    schema: analytics
    description: Outputs of scripts not yet converted. Shrinks to nothing.
    tables:
      - name: daily_events
        loaded_at_field: _loaded_at
        freshness:
          warn_after: {count: 26, period: hour}
      - name: user_summary
      - name: order_facts
```

Zero risk — you've written YAML. In exchange:

- Every downstream dbt model has real lineage back to the script
- `dbt source freshness` tells you when a script silently stops running
- The `legacy` source is a **visible backlog**: its length is your remaining work

That last point is worth the effort on its own. A shrinking list of tables in one
YAML file is the clearest migration progress indicator there is.

## Step 2, atomically

When you convert `daily_events.sql`:

1. Add `models/staging/daily_events.sql`
2. Remove `daily_events` from the `legacy` source
3. Change every `source('legacy', 'daily_events')` to `ref('daily_events')`
4. Disable the script's schedule

Steps 2 and 3 must be **one commit**. If the source is removed while a consumer
still references it, parsing fails; if the source stays while the model exists,
you have two nodes for one table and forked lineage —
[E3](../E-translation/E3-ref-vs-source.md).

Step 4 is separate, and it comes after
[verification](../H-verification/H13-sign-off.md), not with the code change.

## Both writing at once

The dangerous window: script disabled, model enabled — or worse, neither
disabled. Two writers to one table produces interleaved, irreproducible output.

The safe sequence: **shadow first**, then swap.

- Model writes to a shadow dataset while the script keeps production
  ([H5](../H-verification/H5-shadow-mode.md))
- Parity proven
- One change: script off, model points at production
- Verify only dbt is writing ([G1](../G-scheduling/G1-cron-to-dbt-build.md))

Never "run both against production and see which wins".

## Direction of travel

Convert roots first ([I1](I1-conversion-order.md)), so the `legacy` source shrinks
from the upstream end. Each conversion makes the next one easier, because its
inputs are already models.

The alternative — converting marts first — leaves you with a long-lived `legacy`
source and models whose inputs all change later.

## Knowing when you're done

The `legacy` source is empty, and:

```sql
select user_email, destination_table.table_id, max(creation_time) as last_run
from `region-eu`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
where creation_time > timestamp_sub(current_timestamp(), interval 30 day)
  and destination_table.dataset_id = 'analytics'
group by 1, 2
order by 1;
```

Only the dbt service account appears. Anything else is a script still running that
nobody remembered — which is exactly what a 30-day-old inventory misses.

---

Previous: [I1 · Conversion order](I1-conversion-order.md) ·
Next: [I3 · Converting while others depend on the output](I3-converting-with-dependents.md)
