# E2 · Ordering by `ref()` instead of by line number

> **Part E — Statement-level translation** · Sourcing: `CORE✓`
> **The question:** my script runs A then B. Where does that order go?

It doesn't go anywhere. You delete it, and dbt derives it.

## The swap

Your script states order:

```sql
CREATE OR REPLACE TABLE staging.cleaned AS SELECT ... FROM raw.events;
CREATE OR REPLACE TABLE analytics.summary AS SELECT ... FROM staging.cleaned;
```

Your models state *reads*:

```sql
-- models/staging/cleaned.sql
select ... from {{ source('raw', 'events') }}
```

```sql
-- models/analytics/summary.sql
select ... from {{ ref('cleaned') }}
```

Nothing says "run cleaned first". dbt infers it, because `summary` reads
`cleaned`, and a thing must exist before it can be read.

## How the graph is actually built

From `compilation.py`, `link_node` walks each node's recorded dependencies and
adds an edge per one:

```python
def link_node(self, node, manifest):
    self.add_node(node.unique_id)
    for dependency in node.depends_on_nodes:
        if dependency in manifest.nodes:
            self.dependency(node.unique_id, manifest.nodes[dependency].unique_id)
        elif dependency in manifest.sources:
            ...
        else:
            raise GraphDependencyNotFoundError(node, dependency)
```

`depends_on_nodes` is populated from your `ref()` and `source()` calls at parse
time. **The graph is a product of what your models read** — there is no ordering
config, and there is no way to add an edge except by reading something.

## Three consequences worth knowing

**A missing target fails at parse, not at run.** `resolve_ref` raises
`TargetNotFoundError` when the model doesn't exist or is disabled. You find out
before anything executes.

**Cycles are rejected outright.** `link_graph` finishes with:

```python
cycle = self.find_cycles()
if cycle:
    raise CompilationError("Found a cycle: {}".format(cycle))
```

If your scripts have a circular dependency resolved by scheduling — A reads
yesterday's B, B reads today's A — dbt will not accept it. That needs designing
out, not converting. It's usually two concerns sharing one table.

**Reading a table without `ref()` creates no edge.** Hardcode
`analytics.cleaned` instead of `{{ ref('cleaned') }}` and the SQL still works,
but dbt doesn't know the two are related. It may build them in either order, and
in parallel. This is the single most common conversion bug, and it produces
intermittent failures rather than consistent ones — see [E5](E5-finding-hardcoded-names.md).

## What to delete from your orchestration

Once both sides are converted:

- Schedule offsets that exist to space jobs out
- "Wait for job A" sensors between two things dbt now owns
- Retry logic compensating for a race the graph removes
- The wiki page describing the order

Replace all of it with one invocation:

```bash
dbt build --select +summary
```

The `+` prefix means "and everything it depends on". Order is derived, and it
stays correct when someone adds a dependency later — which the offsets never did.

## Self-reference is not a cycle

An incremental model reading its own target looks circular but isn't:

```sql
{% if is_incremental() %}
  where updated_at > (select max(updated_at) from {{ this }})
{% endif %}
```

`{{ this }}` is not `ref()`. It resolves to the model's own relation without
recording a dependency, so no edge is created and no cycle is found. That's the
watermark pattern from [B6](../B-write-patterns/B6-watermark-filter.md), and it's
entirely normal.

---

Previous: [A8 · Estimate conversion risk](../A-assess/A8-estimate-risk.md) ·
Next: [E3 · `ref` vs `source`](E3-ref-vs-source.md)
