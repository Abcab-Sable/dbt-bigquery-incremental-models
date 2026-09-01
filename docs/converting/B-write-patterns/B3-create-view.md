# B3 · `CREATE [OR REPLACE] VIEW` → `materialized='view'`

> **Part B — Write-pattern archetypes** · Sourcing: `SRC`
> **The question:** does a view script convert, and should it stay a view?

Two questions. The conversion is trivial; whether it should remain a view usually
isn't.

## The conversion

```sql
CREATE OR REPLACE VIEW analytics.active_customers AS
SELECT * FROM analytics.customers WHERE status = 'active';
```

```sql
{{ config(materialized='view') }}

select * from {{ ref('customers') }}
where status = 'active'
```

That's it. Note `ref()` rather than the hardcoded name — a view has dependencies
like anything else, and without `ref()` there's no edge
([E5](../E-translation/E5-finding-hardcoded-names.md)).

## Should it stay a view?

Views cost nothing to store and everything to read, every time. The question is
how often it's read and how expensive the underlying query is.

| Situation | Materialization |
| --- | --- |
| Read rarely, cheap query | `view` |
| Read constantly by dashboards | `table` |
| Expensive aggregation, read more than once a day | `table` |
| Thin renaming/filtering layer over one table | `view` |
| Several views stacked on each other | Consider `table` at the bottom |

Stacked views are the case worth hunting for. A view over a view over a view
means the innermost query re-runs for every read of the outermost, and BigQuery
bills all of it. Scripts accumulate these because creating a view is the cheapest
possible action at the time.

## Ephemeral, if it's only an intermediate step

If the view exists solely so another model can read it, and nothing else queries
it, `ephemeral` removes the object entirely and inlines it as a CTE:

```sql
{{ config(materialized='ephemeral') }}
```

Fewer objects, no stale intermediate. But it disappears from the warehouse, so
anything querying it directly breaks — check first with
[A2](../A-assess/A2-map-dependencies.md). More in
[C2](../C-structural/C2-ephemeral-models.md).

## The view-to-table trap

If you convert a model from `view` to `table` or `incremental`, dbt must **drop
the view first**. There's no atomic view-to-table replacement on BigQuery:

```jinja
{% elif existing_relation.is_view %}
    {#-- There's no way to atomically replace a view with a table on BQ --#}
    {{ adapter.drop_relation(existing_relation) }}
```

Between the drop and the create, the relation does not exist. Anything querying
it at that moment fails. Schedule the change accordingly, and tell whoever reads
it — [I5](../I-migration/I5-notifying-consumers.md).

## Authorized views

If the view exists to grant access to a subset of a table without granting the
table, that's an authorized view, and the authorization is a **grant**, not part
of the view definition. Converting the view without carrying the authorization
breaks access silently — [D10](../D-data-movement/D10-grants-authorized-views.md).

---

Previous: [B2 · `CREATE TABLE IF NOT EXISTS` bootstrap](B2-create-if-not-exists.md) ·
Next: [B4 · Materialized views](B4-materialized-views.md)
