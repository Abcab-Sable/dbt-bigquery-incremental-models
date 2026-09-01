# K5 · Rebuilding the script's imperative structure in Jinja

> **Part K — Anti-patterns** · Sourcing: `CRAFT`
> **The question:** Jinja has loops and conditionals. Why not use them to port the procedure?

Because Jinja runs at **compile** time and the script's logic ran at **run** time.
Forcing one into the other produces something that looks like the original and
behaves differently.

## The shape

```sql
{% set regions = run_query("select distinct region from raw.events").columns[0].values() %}

{% for region in regions %}
  {% set row_count = run_query("select count(*) from raw.events where region = '" ~ region ~ "'") %}
  {% if row_count.columns[0].values()[0] > 0 %}
    select * from raw.events where region = '{{ region }}'
    {% if not loop.last %}union all{% endif %}
  {% endif %}
{% endfor %}
```

A loop, a condition, and two queries — all at compile time, all to produce SQL a
`GROUP BY` would have expressed.

## What's wrong

**Warehouse round-trips on every compile.** `dbt compile`, `dbt docs generate`,
CI parsing, every developer's build. Slow, costly, and invisible.

**The dependency isn't in the DAG.** `run_query` reads a table without creating an
edge, so dbt may build things in the wrong order —
[E5](../E-translation/E5-finding-hardcoded-names.md).

**Compiled SQL varies by when you compiled it.** Two developers get different
models from the same code, which makes review and parity checking unreliable.

**It's unreadable.** Nested Jinja generating SQL is harder to follow than either
the script or a plain query.

**String concatenation into SQL.** `'" ~ region ~ "'` is injection waiting to
happen if the values aren't trusted.

## What it should be

Almost always set-based:

```sql
select region, ... from {{ source('raw', 'events') }}
group by region
```

Regions become a column instead of a loop. Empty regions produce no rows, so the
condition disappears. One statement, no compile-time queries, full lineage.

That's [C7](../C-structural/C7-loops.md) — most loops are a `GROUP BY` written the
long way.

## Where Jinja loops are fine

Over a **statically known** list, generating repetitive SQL:

```sql
{% set metrics = ['clicks', 'views', 'purchases'] %}

select
    user_id,
    {% for m in metrics %}
    countif(event_type = '{{ m }}') as {{ m }}_count{{ ',' if not loop.last }}
    {% endfor %}
from {{ ref('events') }}
group by user_id
```

The list is in the code, no queries at compile time, and the generated SQL is
stable. That's templating doing its job.

The line is **`run_query` in a loop or condition**. Once compile-time behaviour
depends on data, you've crossed from templating into programming.

## The narrow legitimate case

Genuine metaprogramming — generating a column list from the schema — is fine, and
`adapter.get_columns_in_relation` is better than raw `run_query`:

```sql
{% set cols = adapter.get_columns_in_relation(ref('events')) %}
```

Structural, not business logic, and it goes through the adapter's caching.

## The honest alternative

If the logic is genuinely procedural and set-based SQL can't express it, the
answer isn't cleverer Jinja. It's:

- Keep it outside dbt and source the output —
  [A7](../A-assess/A7-what-not-to-convert.md)
- Or redesign so the computation belongs somewhere that does iteration well

Both are better than a model nobody can debug. A conversion that produces harder
code than the script has failed even if it runs.

---

Previous: [K4 · `run-operation` as a scheduler](K4-run-operation-as-scheduler.md) ·
Next: [K6 · Porting the bug faithfully](K6-porting-the-bug.md)
