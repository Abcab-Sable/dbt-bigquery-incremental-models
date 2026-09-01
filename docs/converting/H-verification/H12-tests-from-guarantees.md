# H12 · Encoding the script's guarantees as dbt tests

> **Part H — Proving correctness** · Sourcing: `CORE`
> **The question:** the script guaranteed things implicitly. How do I keep those?

Write them as tests. Parity proves the conversion was right *once*; tests keep it
right. This is the step that converts a one-off verification into a standing one.

## Find the implicit guarantees

Scripts assert things without saying so. Each becomes a test:

| The script did | It was guaranteeing | Test |
| --- | --- | --- |
| `MERGE ... ON T.id = S.id` | `id` is unique | `unique` |
| `QUALIFY ROW_NUMBER() ... = 1` | one row per key | `unique` |
| `WHERE x IS NOT NULL` | `x` is never null downstream | `not_null` |
| `DELETE ... WHERE date >= x` then insert | no rows outside the range | a range test |
| `JOIN dim_users` | every user resolves | `relationships` |
| `WHERE status IN (...)` | status is from a known set | `accepted_values` |
| An `ASSERT` statement | exactly what it said | a singular test |

The `MERGE` key is the most valuable and the most commonly skipped. If the script
merged on `order_id`, uniqueness was load-bearing — and after conversion it's
enforced by a config you might get wrong.

## The basics

```yaml
models:
  - name: orders
    columns:
      - name: order_id
        data_tests:
          - unique
          - not_null
      - name: status
        data_tests:
          - accepted_values:
              values: ['pending', 'shipped', 'delivered', 'cancelled']
      - name: customer_id
        data_tests:
          - relationships:
              to: ref('customers')
              field: customer_id
```

`unique` and `not_null` on the `unique_key` are close to mandatory after a
conversion. They're the direct check on the failure modes in
[B8](../B-write-patterns/B8-merge-on-clause-to-unique-key.md) — a composite key
with a nullable column produces duplicates, and this catches it on the first run
rather than the fiftieth.

## Composite keys

Test the combination, not the columns individually:

```yaml
models:
  - name: order_lines
    data_tests:
      - unique_combination_of_columns:
          combination_of_columns: [order_id, line_no]
```

That's `dbt_utils`. Without it, a singular test does the job:

```sql
-- tests/order_lines_unique_key.sql
select order_id, line_no, count(*) as n
from {{ ref('order_lines') }}
group by order_id, line_no
having count(*) > 1
```

A test passes when it returns zero rows.

## `ASSERT` statements convert directly

If the script had explicit checks, they're already tests:

```sql
ASSERT (SELECT COUNT(*) FROM analytics.daily_events WHERE event_count < 0) = 0
    AS 'negative event counts';
```

```sql
-- tests/daily_events_no_negative_counts.sql
select * from {{ ref('daily_events') }} where event_count < 0
```

Better than the original: it runs as part of `dbt build`, it's named, and a
failure blocks downstream models rather than aborting a script mid-way.

## Tests that encode the conversion's risks

Beyond the script's guarantees, add tests for what conversion introduced:

**Freshness of the partition that should be current** — catches a model silently
not writing:

```sql
-- tests/daily_events_has_yesterday.sql
select 1
from (select max(event_date) as m from {{ ref('daily_events') }})
where m < date_sub(current_date(), interval 1 day)
```

**Row counts within a plausible band** — catches an empty run or a runaway join:

```sql
-- tests/daily_events_plausible_volume.sql
select event_date, count(*) as n
from {{ ref('daily_events') }}
where event_date >= date_sub(current_date(), interval 7 day)
group by event_date
having count(*) < 100 or count(*) > 10000000
```

Crude, and it will catch a genuine outage before anyone notices the dashboard.

## Severity

Not everything should block the build:

```yaml
- name: customer_id
  data_tests:
    - relationships:
        to: ref('customers')
        field: customer_id
        config:
          severity: warn
```

Use `error` for things that mean the data is wrong, `warn` for things that mean
it's odd. A project where every test is `error` gets its failures ignored, which
is worse than having fewer tests.

## Run them as part of the build

```bash
dbt build --select daily_events
```

`build` runs models and their tests in dependency order, so a failing test stops
downstream models from consuming bad data. `dbt run` followed by `dbt test` does
not — by the time the test fails, everything downstream has already been built
from it.

For a converted model, `build` is the right verb.

---

Previous: [H11 · Differences that should exist](H11-differences-that-should-exist.md) ·
Next: [H13 · Sign-off](H13-sign-off.md)
