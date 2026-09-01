# E5 · Finding every hardcoded table name

> **Part E — Statement-level translation** · Sourcing: `CRAFT`
> **The question:** how do I know I converted them all?

A missed hardcoded name doesn't error. The SQL is valid, the table exists, the
model runs. You just lost an edge in the graph — and with it, the ordering
guarantee that was the point of converting.

This is the most common conversion bug, and it's mechanical to prevent.

## Why it's worse than a normal bug

Missing a `ref()` doesn't fail. It produces a model that:

- may be built **before** its dependency, because dbt sees no reason not to
- may be built **in parallel** with it
- shows no edge in the lineage graph
- works fine most days, because the ordering usually happens to be right

Intermittent failures with no obvious cause, appearing weeks later when the
project grows enough to change the build order.

## The sweep

Find anything that looks like a qualified table name in your models:

```bash
grep -rnE '`?[a-z0-9_-]{3,}`?\.`?[a-z0-9_]+`?\.`?[a-z0-9_]+`?' models/ \
  | grep -v '{{' 
```

The `grep -v '{{'` drops lines already templated. What remains is your list.

Then the two-part form, for same-project references:

```bash
grep -rnE '\b(from|join)\s+`?[a-z0-9_]+`?\.`?[a-z0-9_]+`?' models/ | grep -v '{{'
```

## The authoritative check

Greps miss things — names built by concatenation, references inside macros,
wildcards. Ask dbt what it thinks the model depends on, and compare with what
BigQuery actually read:

```bash
dbt compile --select my_model
```

```sql
select referenced_tables
from `region-eu`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
where creation_time > timestamp_sub(current_timestamp(), interval 1 hour)
  and destination_table.table_id = 'my_model'
order by creation_time desc
limit 1;
```

**Every table in `referenced_tables` should correspond to a `ref()` or
`source()` in the model.** Anything present in BigQuery's list but absent from
your model's dependencies is a missing edge.

This catches everything the greps don't, because it's observed behaviour rather
than pattern matching.

## The lineage eyeball

After converting a batch, look at the graph:

```bash
dbt docs generate && dbt docs serve
```

A model with **no parents** that obviously should have some is the tell. So is a
cluster of models that ought to be a chain but renders as unconnected islands.

Quicker than reading SQL, and it catches the case where you converted the
reference but pointed it at the wrong model.

## The cases greps won't catch

| Pattern | Why it hides |
| --- | --- |
| `EXECUTE IMMEDIATE` with a built name | The name doesn't exist as text — [C10](../BACKLOG.md#part-c--structural-archetypes) |
| Wildcards (`events_*`) | Not a single table — [D4](../BACKLOG.md#part-d--data-movement-ddl-and-metadata) |
| Reads through a view | The view is the dependency; what it reads is invisible |
| Names inside a macro | Grep `macros/` too |
| Names in a hook | Hooks are SQL and can hardcode. Grep configs as well |

That last one matters and is easy to forget — a `post_hook` referencing another
table creates no edge either.

## Make it a check

Once converted, keep it converted:

```bash
grep -rnE '`[a-z0-9_-]+\.[a-z0-9_]+\.[a-z0-9_]+`' models/ && exit 1 || exit 0
```

A crude CI step, and it stops the pattern coming back the next time someone is in
a hurry.

---

Previous: [E4 · Cross-project references](E4-cross-project-references.md) ·
Next: [E6 · Hardcoded dates and backfill parameters](E6-hardcoded-dates.md)
