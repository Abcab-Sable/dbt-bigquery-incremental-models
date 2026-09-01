# C11 · `CREATE TEMP FUNCTION`

> **Part C — Structural archetypes** · Sourcing: `CRAFT`
> **The question:** my script defines a UDF at the top. Where does it live now?

Three options: a Jinja macro, a persistent UDF, or a pre-hook. The macro is
usually right for SQL logic; the persistent UDF for JavaScript.

## The pattern

```sql
CREATE TEMP FUNCTION parse_domain(email STRING) AS (
    LOWER(SPLIT(email, '@')[SAFE_OFFSET(1)])
);

SELECT user_id, parse_domain(email) AS domain FROM raw.users;
```

The function exists for the session, then disappears.

## Option 1: a Jinja macro (usually right)

```sql
{% macro parse_domain(column) %}
    lower(split({{ column }}, '@')[safe_offset(1)])
{% endmacro %}
```

```sql
select user_id, {{ parse_domain('email') }} as domain
from {{ source('raw', 'users') }}
```

The macro expands at compile time, so the SQL is inlined. No function object
exists, nothing to create or manage, and it works in every environment
automatically.

**Advantages:** version-controlled, reusable across models, visible in compiled
output, no dependency on session state or deployment order.

**Limitation:** it's text substitution, not a function. It can't recurse, and
complex logic produces long inlined SQL. For most script UDFs — a bit of string
handling, a case expression, a date calculation — that's fine.

## Option 2: a persistent UDF

When the logic is genuinely function-shaped, especially JavaScript UDFs which
have no Jinja equivalent:

```sql
CREATE OR REPLACE FUNCTION `analytics.udf.parse_domain`(email STRING)
RETURNS STRING AS (
    LOWER(SPLIT(email, '@')[SAFE_OFFSET(1)])
);
```

Deploy it once, then reference it:

```sql
select user_id, `{{ target.project }}.udf.parse_domain`(email) as domain
from {{ source('raw', 'users') }}
```

**The problem is deployment.** dbt doesn't manage function objects. Options:

- **`on-run-start`** — recreated at the top of every run. Simple, and adds a
  statement per invocation rather than per model.
- **A `run-operation`** in your deploy pipeline. Cleaner separation, but the
  function's existence becomes a deployment prerequisite nobody sees in the DAG.
- **Terraform or equivalent.** Right answer if you already manage infrastructure
  that way.

Whichever you pick, the function must exist in **every** environment. A model
referencing a UDF that's only in prod fails in dev with a confusing error.

## Option 3: a pre-hook

```sql
{{ config(pre_hook="{{ create_temp_function_parse_domain() }}") }}
```

Works — the function and the model share a session. Legitimate as a stopgap, and
listed in [F8](../F-hooks/F8-pre-hook-patterns.md).

But it runs per model, so ten models using the function create it ten times, and
the dependency is hidden in a config string. Prefer options 1 or 2.

## Choosing

| The UDF is | Use |
| --- | --- |
| SQL, simple | A Jinja macro |
| SQL, used by many models | A Jinja macro |
| JavaScript | A persistent UDF — no macro equivalent |
| Complex, recursive, or genuinely a function | A persistent UDF |
| A stopgap during migration | A pre-hook, with a ticket |

## Environment-qualify the name

If you go persistent, don't hardcode the project:

```sql
-- wrong: reads prod from dev
`acme-prod.udf.parse_domain`(email)

-- right
`{{ target.project }}.udf.parse_domain`(email)
```

Same reasoning as [E4](../E-translation/E4-cross-project-references.md), and the
same failure — dev silently depending on prod.

## Test it

A macro can be tested by building a small model that exercises it and asserting
the output. A persistent UDF can be tested the same way. Script UDFs typically had
no tests at all, so this is a free improvement — and worth doing, because a subtle
change in the logic during conversion is otherwise invisible.

---

Previous: [C10 · `EXECUTE IMMEDIATE` and dynamic SQL](C10-dynamic-sql.md) ·
Next: [C12 · Procedures calling procedures](C12-nested-procedures.md)
