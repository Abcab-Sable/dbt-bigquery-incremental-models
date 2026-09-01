# D11 · Policy tags and row-level security

> **Part D — Data movement, DDL, metadata** · Sourcing: `CRAFT`
> **The question:** our table has column-level tags and row filters. Do they survive?

Not automatically. Both attach to the relation, and dbt recreates relations. This
is the highest-consequence hidden state in a conversion, because losing it means
exposing data rather than losing access.

## What they are

**Policy tags** — column-level access control. A column tagged
`pii/email` is readable only by principals granted the tag's Fine-Grained Reader
role. Applied per column.

**Row-level security** — `CREATE ROW ACCESS POLICY` filters which rows a principal
sees. Applied per table.

Neither is covered by the `grants` config.

## Find them before you convert

Policy tags:

```sql
select column_name, policy_tags
from `project.analytics.INFORMATION_SCHEMA.COLUMN_FIELD_PATHS`
where table_name = 'customers'
  and policy_tags is not null;
```

Row access policies:

```sql
select *
from `project.analytics.INFORMATION_SCHEMA.ROW_ACCESS_POLICIES`
where table_name = 'customers';
```

Anything returned is state that must be reapplied after every recreate, or
deliberately retired.

## The failure direction matters

Losing a grant means someone **can't** read data — annoying, visible, quickly
reported.

Losing a policy tag means someone **can** read data they shouldn't. Nobody
reports that. It surfaces in an audit, months later.

So this belongs in the pre-conversion checklist, not the post-cutover one, and it
raises the model's risk score in [A8](../A-assess/A8-estimate-risk.md)
regardless of how simple the SQL is.

## When they get lost

- `--full-refresh` where partitioning changed ⇒ **drop and recreate**
  ([D6](D6-partitioning-ddl.md))
- `view` → `table` conversion ⇒ the view is dropped first
  ([B3](../B-write-patterns/B3-create-view.md))
- Any manual rebuild during development that gets promoted

The first is the likely one during a conversion, because changing `partition_by`
is a common part of consolidating a script's table.

## Reapplying them

dbt has no native support. Options, best first:

**Terraform or your IAM tooling.** Policy tags are governance configuration; they
belong with your other governance configuration, and they get reapplied
independently of dbt runs.

**A `run-operation` in the deploy pipeline.** Explicit, ordered after the dbt
build.

**An `on-run-end` hook.** Keeps it with dbt, runs once per invocation. Workable,
but it means your access controls depend on a dbt run completing.

**A per-model post-hook.** Runs before `apply_grants`
([F4](../F-hooks/F4-where-hooks-run.md)) and on every build. Least good, but
sometimes the only option available quickly.

## Column-level tags argue for stable column names

A policy tag is attached to a column by name. Renaming a column during conversion
— even to something clearer — drops the tag with it.

Another reason to keep the script's column names through the conversion and tidy
separately ([K11](../K-antipatterns/K11-convert-and-optimise.md)). Here the cost of
renaming isn't a confusing diff; it's an unprotected PII column.

## Verify with a real principal

`INFORMATION_SCHEMA` will show you the policies. It won't tell you whether the
effective access is right.

After cutover, have someone who **should not** see the protected column or rows
confirm they can't:

```sql
select email from analytics.customers limit 1;   -- should error for them
```

That's the only check that actually validates the control. Do it as part of
[H13](../H-verification/H13-sign-off.md), and record who confirmed it.

---

Previous: [D10 · Grants and authorized views](D10-grants-authorized-views.md) ·
Next: [D12 · `ASSERT` data-quality gates](D12-assert-gates.md)
