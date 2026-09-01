# F11 · post-hook vs the `grants` config

> **Part F — Hooks** · Sourcing: `SRC`
> **The question:** my script grants access at the end. Post-hook or config?

The config. A post-hook `GRANT` races dbt's own grant handling, and dbt runs
second.

## The ordering, from source

In the BigQuery incremental materialization:

```jinja
{{ run_hooks(post_hooks) }}                                    ← your hook

{% set target_relation = this.incorporate(type='table') %}
{% set should_revoke = should_revoke(existing_relation, full_refresh_mode) %}
{% do apply_grants(target_relation, grant_config, should_revoke) %}   ← dbt's grants
```

Your post-hook runs **first**. `apply_grants` runs **after**, with
`should_revoke` computed from whether this was a fresh build or a full refresh.

Same order in `table.sql` — post-hooks at line 37, `apply_grants` at line 40.

## Why that's a problem

`apply_grants` doesn't merely add grants. When `should_revoke` is true it
reconciles the relation's grants **to match the config** — which means revoking
anything not in it.

So a post-hook granting `roles/bigquery.dataViewer` to a group not present in the
`grants` config can be granted by your hook and then revoked by dbt, in the same
run, seconds apart.

The symptom is maddening: access appears to work sometimes, and the run log shows
both statements succeeding. Nothing errors, because both did exactly what they
were asked.

## The fix

```sql
{{ config(
    materialized='incremental',
    grants={
        'roles/bigquery.dataViewer': ['group:analysts@acme.com'],
        'roles/bigquery.dataEditor': ['serviceAccount:etl@acme.iam.gserviceaccount.com']
    }
) }}
```

Or in `dbt_project.yml` for a whole folder:

```yaml
models:
  my_project:
    marts:
      +grants:
        roles/bigquery.dataViewer: ['group:analysts@acme.com']
```

dbt now owns grants end to end. They're declarative, visible in the manifest,
diffable in review, and reconciled every run rather than accumulating.

## Converting a script's `GRANT` statements

Read the live state rather than the script — hand-granted access is exactly the
[hidden state](../A-assess/A5-hidden-state.md) that won't survive:

```sql
select privilege_type, grantee
from `project.dataset.INFORMATION_SCHEMA.OBJECT_PRIVILEGES`
where object_name = 'daily_events';
```

Compare that to the script. Differences are grants someone applied by hand, and
you have to decide deliberately whether each one is kept — because once dbt owns
the config, anything absent from it gets revoked on the next full refresh.

That decision is easy to miss and expensive to discover. Do it during
[A5](../A-assess/A5-hidden-state.md), not after cutover.

## Inheritance is additive per role

Grants set at project level and model level combine per role. If you need a model
to have *only* the grants it declares, set them at the model and check nothing
broader applies from `dbt_project.yml`.

## When a hook is still needed

Authorized views, row-level access policies, and policy tags aren't covered by
the `grants` config. Those may need a hook or an out-of-band process —
[D11](../D-data-movement/D11-policy-tags-rls.md).

If you do need one, be aware it runs before `apply_grants`, and make sure the two
aren't operating on the same principal-and-role pair. Different concerns, no
race. Same concern, and you're back to the problem above.

---

Previous: [F10 · post-hook patterns worth keeping](F10-post-hook-patterns.md) ·
Next: [F12 · post-hook: table options](F12-post-hook-table-options.md)
