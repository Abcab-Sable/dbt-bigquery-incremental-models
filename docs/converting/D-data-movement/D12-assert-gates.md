# D12 · `ASSERT` data-quality gates → dbt tests

> **Part D — Data movement, DDL, metadata** · Sourcing: `CRAFT`
> **The question:** my script asserts conditions and aborts. What replaces it?

dbt tests, and the replacement is better in every respect — they run in dependency
order, block downstream models, and report uniformly.

## The pattern

```sql
ASSERT (SELECT COUNT(*) FROM analytics.daily_events WHERE event_count < 0) = 0
    AS 'negative event counts found';

ASSERT (SELECT COUNT(DISTINCT event_date) FROM analytics.daily_events
        WHERE event_date = CURRENT_DATE()) > 0
    AS 'no data for today';
```

## The conversion

A dbt test is a query that **returns rows when something is wrong**:

```sql
-- tests/daily_events_no_negative_counts.sql
select * from {{ ref('daily_events') }} where event_count < 0
```

```sql
-- tests/daily_events_has_today.sql
select 1
from (select count(*) as n from {{ ref('daily_events') }}
      where event_date = current_date())
where n = 0
```

Zero rows ⇒ pass. That's the whole convention.

## Prefer built-in tests where they fit

Many assertions are standard:

```yaml
models:
  - name: daily_events
    columns:
      - name: event_id
        data_tests: [unique, not_null]
      - name: status
        data_tests:
          - accepted_values:
              values: ['pending', 'shipped', 'delivered']
      - name: user_id
        data_tests:
          - relationships:
              to: ref('users')
              field: user_id
```

Less code, better failure messages, and they show in the docs site as
documentation of the guarantees.

## Why this beats `ASSERT`

**It runs in dependency order.** `dbt build` runs a model's tests immediately
after the model and **before** its dependents. A failure stops bad data
propagating.

An `ASSERT` at the end of a script fired after everything had already been
written, and after downstream jobs may have read it.

**Failures don't abort everything.** A failed test blocks that model's
descendants; unrelated branches continue. The script's `ASSERT` killed the whole
run.

**Severity is adjustable:**

```yaml
- name: user_id
  data_tests:
    - relationships:
        to: ref('users')
        field: user_id
        config:
          severity: warn
```

`ASSERT` had one severity: fatal. Most scripts therefore assert only the things
worth dying for, and skip the rest. Tests let you record the softer expectations
too.

**Failures are inspectable:**

```yaml
- unique:
    config:
      store_failures: true
```

The failing rows land in a table you can query. `ASSERT` gave you a message.

## Use `build`, not `run` then `test`

```bash
dbt build --select daily_events
```

`dbt run` followed by `dbt test` runs *every* model first, then tests. By the time
a test fails, everything downstream has already been built from bad data — which
is the behaviour you were converting away from.

For a converted model, `build` is the right verb. Worth putting in the runbook
explicitly, because `run` is the more familiar command.

## Convert the implicit assertions too

The script's explicit `ASSERT`s are the easy half. The implicit guarantees —
what the `MERGE` key assumed, what a `WHERE NOT NULL` protected — are the more
valuable half, and they're covered in
[H12](../H-verification/H12-tests-from-guarantees.md).

A conversion is the one time you'll be reading the script closely enough to find
them.

---

Previous: [D11 · Policy tags and row-level security](D11-policy-tags-rls.md) ·
Next: [D13 · Notification side-effects](D13-notifications.md)
