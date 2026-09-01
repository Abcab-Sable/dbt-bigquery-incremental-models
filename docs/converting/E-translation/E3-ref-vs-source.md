# E3 · `ref` vs `source`, and never both

> **Part E — Statement-level translation** · Sourcing: `CORE✓`
> **The question:** which of my script's input tables become sources?

One rule: **`ref()` for tables dbt builds, `source()` for tables it doesn't.**
That's the whole distinction, and everything else follows from it.

## Sources are graph roots by construction

Look at how the two are linked in `compilation.py`:

```python
def link_graph(self, manifest):
    for source in manifest.sources.values():
        self.add_node(source.unique_id)          # added, never linked
    for node in manifest.nodes.values():
        self.link_node(node, manifest)           # linked — edges from its deps
    ...
```

Sources get `add_node`. Models get `link_node`, which walks their dependencies
and creates edges. **A source has no outgoing dependency edges because dbt never
looks for any.**

So declaring something a source is an assertion: *dbt does not build this, and
nothing upstream of it is dbt's concern.* It's a boundary marker, not just a
naming convention.

## Mapping your script's inputs

Go through the input tables from [A2](../A-assess/A2-map-dependencies.md):

| The table is | Use |
| --- | --- |
| Loaded by Fivetran, Airbyte, a stream, anything external | `source()` |
| Produced by a script you haven't converted yet | `source()` — for now |
| Produced by a model in this project | `ref()` |
| Produced by a script you're converting in the same change | `ref()`, once both exist |
| This model's own target (watermark) | `{{ this }}` — neither |

## The "for now" case is the useful one

A script you haven't converted still produces a table. Declare its output as a
source and downstream models get lineage and freshness checks immediately,
without converting anything:

```yaml
sources:
  - name: legacy
    schema: analytics
    tables:
      - name: daily_events
        description: >
          Produced by daily_events.sql (scheduled query, 03:00 UTC).
          Converting in DATA-2288 — becomes ref() then.
        loaded_at_field: _loaded_at
        freshness:
          warn_after: {count: 26, period: hour}
```

This is the strangler pattern's foothold ([I2](../I-migration/I2-strangler-pattern.md))
and the middle ground from [A7](../A-assess/A7-what-not-to-convert.md). Highest
value per unit of risk in the whole migration.

When you convert that script, the source declaration is deleted and every
`source('legacy', 'daily_events')` becomes `ref('daily_events')`. Do it in one
change so nothing points at both.

## Never both

The failure: a table declared as a source *and* built by a model. Now you have
two nodes for one physical table, and lineage silently forks. Half your project
thinks it's a root; the other half thinks it's derived.

Symptoms are ordering that looks right until it isn't, and a DAG that doesn't
match reality. Nothing errors, because as far as dbt is concerned these are two
different objects that happen to share a name.

When you convert a script, grep for the old source name and make sure every
reference moved:

```bash
grep -rn "source('legacy', 'daily_events')" models/
```

Zero results, or the conversion is half-done.

## Don't source your own output

An incremental model's watermark reads its own target. Use `{{ this }}`, not a
source pointing at the same table. Sourcing it creates a phantom node and
implies dbt doesn't build something it plainly does.

---

Previous: [E2 · Ordering by `ref()`](E2-ordering-by-ref.md) ·
Next: [E4 · Cross-project references](E4-cross-project-references.md)
