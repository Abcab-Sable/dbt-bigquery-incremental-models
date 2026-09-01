# K10 · No tests, because the script had none

> **Part K — Anti-patterns** · Sourcing: `CRAFT`
> **The question:** the script had no tests and it was fine. Why add them now?

Because the script was fine for reasons that stop applying. Converting to
incremental introduces failure modes that only tests catch, and "it worked before"
is evidence about a different system.

## Why the script got away with it

A `CREATE OR REPLACE` script cannot drift. It recomputes from source every run, so
if the SQL is right the output is right. No accumulated state, nothing to
corrupt.

An incremental model accumulates. Every silent failure in this documentation —
[empty partitions](../B-write-patterns/B14-when-the-range-can-empty.md),
[duplicates](../B-write-patterns/B8-merge-on-clause-to-unique-key.md),
[dropped columns](../D-data-movement/D8-add-column-migrations.md) — is a
divergence between what the model holds and what it should.

The script had no state to be wrong about. Your model does.

## The minimum

Three tests, and they take five minutes:

```yaml
models:
  - name: daily_events
    columns:
      - name: event_id
        data_tests: [unique, not_null]
      - name: event_date
        data_tests: [not_null]
```

`unique` on the key is the single highest-value test after a conversion. It
catches duplicate accumulation on the next run rather than the fiftieth, and
duplicates are the most common silent failure.

`not_null` on the partition column catches rows that would
[never be replaced](../B-write-patterns/B14-when-the-range-can-empty.md) because
of `IGNORE NULLS`.

## Then the script's implicit guarantees

The script asserted things without saying so —
[H12](../H-verification/H12-tests-from-guarantees.md):

| The script did | Test |
| --- | --- |
| `MERGE ON id` | `unique` on `id` |
| `QUALIFY ROW_NUMBER() = 1` | `unique` |
| `WHERE x IS NOT NULL` | `not_null` |
| `JOIN dim_users` | `relationships` |
| `WHERE status IN (...)` | `accepted_values` |
| An `ASSERT` | A singular test |

Each was load-bearing. After conversion they're enforced by config you might have
got wrong, so assert them.

## Don't test everything

The opposite failure. A project where every column has four tests produces slow
builds and alerts nobody reads, and the important failures hide among the noise.

Test:

- The **key**, always
- The **partition column**, always
- Things a **consumer would notice** being wrong
- Things the script **implicitly guaranteed**
- **Volume plausibility**, as a warning

Skip: cosmetic constraints, columns nobody uses, anything already guaranteed by
the source.

Use severity deliberately — `error` for wrong data, `warn` for odd data
([J4](../J-operating/J4-alerting.md)).

## Run them with `build`

```bash
dbt build --select daily_events
```

`dbt run` then `dbt test` builds everything first, so by the time a test fails the
bad data has propagated downstream. `build` runs each model's tests before its
dependants — [D12](../D-data-movement/D12-assert-gates.md).

## The objection, answered

> *"We'd be testing things nobody has ever seen fail."*

The script couldn't fail that way. The model can, and it will fail **silently** —
the whole point of this documentation is that green runs are not evidence.

A `unique` test is one line of YAML against a class of failure that otherwise
surfaces as a quiet number drift someone notices in three months.

## Tests aren't the whole answer

They catch violations of stated rules. They don't catch a model that has quietly
diverged from what a full rebuild would produce — for that you need
[reconciliation](../J-operating/J3-scheduled-reconciliation.md).

Both, ideally. Tests are the cheap continuous layer; reconciliation is the
periodic thorough one.

---

Previous: [K9 · Ephemeral overuse](K9-ephemeral-overuse.md) ·
Next: [K11 · Converting and optimising together](K11-convert-and-optimise.md)
