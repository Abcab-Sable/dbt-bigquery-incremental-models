# D10 · Grants and authorized views

> **Part D — Data movement, DDL, metadata** · Sourcing: `SRC`
> **The question:** my script grants access at the end. Config or hook?

The `grants` config — [F11](../F-hooks/F11-grants-vs-post-hook.md) has the
mechanism and the ordering trap. This page is the conversion work: finding what
access actually exists, and handling authorized views, which `grants` doesn't
cover.

## Read the live state first

The script's `GRANT` statements are not the whole picture. Access gets added by
hand, and hand-granted access is the classic
[hidden state](../A-assess/A5-hidden-state.md):

```sql
select privilege_type, grantee
from `project.analytics.INFORMATION_SCHEMA.OBJECT_PRIVILEGES`
where object_name = 'daily_events';
```

Compare that to the script. Differences are grants someone applied through the
console or `bq`, and each needs a decision.

## The decision that's easy to miss

Once dbt owns grants, `apply_grants` **reconciles** the relation to match the
config when `should_revoke` is true. Anything present on the table but absent from
your config gets **revoked**.

So a grant you forget to carry across doesn't quietly persist — it gets removed on
the next full refresh, and someone loses access.

Go through the live list explicitly:

```
Grants on analytics.daily_events at baseline:
  dataViewer  group:analysts@acme.com       → carried to config
  dataViewer  group:finance@acme.com        → carried to config
  dataViewer  user:someone@acme.com         → DROPPED, left in 2024, confirmed w/ IT
  dataEditor  serviceAccount:etl@...        → carried to config
```

That list is a deliverable of [A5](../A-assess/A5-hidden-state.md), and skipping
it produces an access incident at cutover.

## The config

```sql
{{ config(
    grants={
        'roles/bigquery.dataViewer': ['group:analysts@acme.com', 'group:finance@acme.com'],
        'roles/bigquery.dataEditor': ['serviceAccount:etl@acme.iam.gserviceaccount.com']
    }
) }}
```

Or per folder, which is usually better than per model:

```yaml
models:
  my_project:
    marts:
      +grants:
        roles/bigquery.dataViewer: ['group:analysts@acme.com']
```

Grant to **groups**, not individuals. A converted project is a good moment to fix
that if the script granted to named users — but do it as a deliberate, separate
change, and tell people.

## Authorized views: `grants` doesn't cover them

An authorized view lets a view read a table that its *users* cannot read directly.
The authorization lives on the **source dataset**, not on the view — so it isn't a
grant on the model at all.

If the script created a view and authorized it, converting the view
([B3](../B-write-patterns/B3-create-view.md)) does **not** carry the
authorization. The view will build and then fail for its users with a permissions
error on the underlying table.

Check for it before converting:

```sql
select * from `project.raw.INFORMATION_SCHEMA.SCHEMATA_OPTIONS`
where option_name = 'authorized_views';
```

Handling options:

- **Terraform or `bq update`**, managed outside dbt. Cleanest, and it's dataset
  configuration rather than model configuration.
- **A `run-operation`** in your deploy pipeline.
- **An `on-run-end` hook**, if it must live with dbt.

Whichever you choose, the authorization must be re-applied when the view is
**recreated** — and a `view` → `table` change drops the relation first
([B3](../B-write-patterns/B3-create-view.md)), which invalidates it.

## Row-level security and policy tags

Also not covered by `grants` — [D11](D11-policy-tags-rls.md). Same warning: they
attach to the table or its columns, and a drop-and-recreate loses them.

That makes them a specific risk when changing `partition_by`, which forces a drop
([D6](D6-partitioning-ddl.md)).

## Test it after cutover

Don't assume. Verify the access someone actually has:

```sql
select privilege_type, grantee
from `project.analytics.INFORMATION_SCHEMA.OBJECT_PRIVILEGES`
where object_name = 'daily_events'
order by grantee;
```

Compare against the baseline list. Then have one real user from each group confirm
they can still query it — the check that catches authorized views and policy tags,
which the privileges view won't show.

---

Previous: [D9 · Column type changes](D9-column-type-changes.md) ·
Next: [D11 · Policy tags and row-level security](D11-policy-tags-rls.md)
