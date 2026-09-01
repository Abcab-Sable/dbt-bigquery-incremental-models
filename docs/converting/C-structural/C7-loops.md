# C7 · `WHILE` / `LOOP` iteration

> **Part C — Structural archetypes** · Sourcing: `CRAFT`
> **The question:** my procedure loops. What's the dbt equivalent?

Usually none, and that's the right answer rather than a limitation. Almost every
loop in a SQL script is one of three things, and only one of them stays a loop.

## Work out which loop you have

**A set operation written iteratively.** The commonest by far:

```sql
FOR record IN (SELECT user_id FROM users) DO
    INSERT INTO summary
    SELECT user_id, COUNT(*) FROM events WHERE user_id = record.user_id;
END FOR;
```

That's a `GROUP BY`:

```sql
select user_id, count(*) as n
from {{ ref('events') }}
group by user_id
```

The loop existed because someone thought row-at-a-time. It's also orders of
magnitude slower. Converting this is a straight win — rewrite it set-based and
the loop disappears.

**A loop over a known list.** Iterating tables, regions, or tenants:

```sql
FOR tbl IN (SELECT table_name FROM INFORMATION_SCHEMA.TABLES WHERE ...) DO
    INSERT INTO combined SELECT * FROM tbl;
END FOR;
```

That's a `UNION ALL`, and Jinja can generate it at compile time:

```sql
{% set regions = ['eu', 'us', 'apac'] %}

{% for region in regions %}
select '{{ region }}' as region, * from {{ source('raw', 'events_' ~ region) }}
{% if not loop.last %}union all{% endif %}
{% endfor %}
```

Jinja loops are **compile-time** — they generate SQL text, not runtime iteration.
That's fine when the list is known without querying. If the list must come from
the data, you're back to `run_query` and its costs
([C6](C6-if-branching.md)).

For wildcard tables specifically, there's a better answer —
[D4](../D-data-movement/D4-wildcard-tables.md).

**Genuine sequential dependency.** Each iteration depends on the last —
running balances, recursive hierarchies, state machines:

```sql
WHILE remaining > 0 DO
    SET balance = balance + (SELECT amount FROM ... WHERE seq = i);
    ...
END WHILE;
```

This one doesn't convert. And it usually shouldn't — see below.

## Sequential dependencies are usually window functions

Before concluding you need a loop, check whether it's actually a window function:

```sql
select
    account_id,
    txn_date,
    amount,
    sum(amount) over (
        partition by account_id
        order by txn_date
        rows between unbounded preceding and current row
    ) as running_balance
from {{ ref('transactions') }}
```

Running totals, gaps and islands, sessionisation, first/last per group — all of
these look sequential and are all set-based. A large fraction of "genuinely
iterative" scripts fall here, and the window version is both simpler and much
faster.

Recursive hierarchies have `WITH RECURSIVE` on BigQuery, which handles most
parent-child traversal.

## When it really is iterative

If after all that it's still a loop — genuine state carried between iterations
that no window function expresses — you have two honest options:

**Keep it outside dbt.** A stored procedure or script, invoked by your
orchestrator, with its output declared as a `source`. dbt gets lineage; the
procedure keeps doing what it does. This is the middle ground from
[A7](../A-assess/A7-what-not-to-convert.md), and it's the right answer more often
than people like.

**Rethink the requirement.** Iterative logic in a warehouse is usually a sign
the computation belongs elsewhere — an application, a stream processor, a
purpose-built job.

## What not to do

**Don't rebuild the loop in Jinja with `run_query`.** You get compile-time
warehouse round-trips, a model nobody can debug, and iteration that isn't
visible in the DAG. That's [K5](../BACKLOG.md#part-k--anti-patterns) — imperative
structure rebuilt in templating, and it's worse than the procedure was.

**Don't use `run-operation` as a loop host** and call it converted. A
`run-operation` that loops is your script with extra steps and less visibility —
[K4](../BACKLOG.md#part-k--anti-patterns).

## Flag it early

A loop is a high-difficulty signal in [A8](../A-assess/A8-estimate-risk.md). Work
out which of the three kinds it is during assessment, not halfway through the
conversion — the answer changes whether this is an afternoon or a redesign.

---

Previous: [C6 · `IF` / `ELSEIF` branching](C6-if-branching.md) ·
Next: [C8 · `EXCEPTION WHEN ERROR`](C8-exception-handling.md)
