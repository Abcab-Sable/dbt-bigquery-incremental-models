# J4 · Alerting: test failures vs run failures

> **Part J — Operating it afterwards** · Sourcing: `CORE✓`
> **The question:** what should page someone, and what should just be a ticket?

They're different signals and deserve different responses. dbt distinguishes them
in its statuses; your alerting should too.

## The statuses

From dbt-core, `RunStatus`:

```
success · error · skipped · partial success · no-op · reused
```

And `TestStatus`:

```
pass · error · fail · warn · skipped
```

The distinction that matters most: a test **`fail`** means the test ran and found
rows — your data is wrong. A test **`error`** means the test query itself broke —
your test is wrong. Those need different people and different urgency.

`FreshnessStatus` adds `pass · warn · error · runtime error` for
[source freshness](J6-freshness-checks.md).

## What each means

| Status | Meaning | Response |
| --- | --- | --- |
| Run `error` | A model failed to build | **Page.** Nothing downstream is fresh |
| Run `skipped` | An ancestor failed | Not itself an alert — find the ancestor |
| Run `partial success` | Some microbatches failed | Investigate; may be retryable |
| Test `fail` | Data violates an assertion | Depends on severity and the test |
| Test `error` | The test query broke | Ticket. A broken test is not bad data |
| Test `warn` | Something odd, not blocking | Ticket, or a digest |
| Freshness `error` | Upstream stopped arriving | **Page.** Usually not your bug |

Alerting on `skipped` is the common mistake — one failure produces a cascade of
skips and a wall of alerts pointing at the wrong models.

## Use severity deliberately

```yaml
models:
  - name: orders
    columns:
      - name: order_id
        data_tests:
          - unique          # error — duplicates are wrong
          - not_null
      - name: customer_id
        data_tests:
          - relationships:
              to: ref('customers')
              field: customer_id
              config:
                severity: warn      # orphans happen; not urgent
```

A project where everything is `error` gets its alerts ignored, which is worse than
having fewer tests. Reserve `error` for "the data is wrong and someone will act on
it".

`warn_if` / `error_if` let you scale by volume:

```yaml
- unique:
    config:
      severity: error
      warn_if: ">0"
      error_if: ">100"
```

## Alert from artefacts, not hooks

Read `run_results.json` after the invocation, or use your orchestrator's failure
callback. Don't try to notify from inside a model — a post-hook doesn't run when
the model fails ([F16](../F-hooks/F16-hooks-and-failure.md)), which is precisely
when you want to hear about it.

Status ⇒ orchestrator. Content ⇒ `on-run-end`. Same split as
[D13](../D-data-movement/D13-notifications.md).

## The alert that matters most isn't a failure

Everything in this documentation fails **silently**. The most valuable alert you
can build isn't on run status at all — it's on
[reconciliation drift](J3-scheduled-reconciliation.md), which is the only signal
that catches an empty partition or an accumulating duplicate.

A pipeline that is green every day and wrong for a month will not page anyone. Add
that check deliberately.

## Route by owner

```yaml
models:
  - name: daily_events
    config:
      group: analytics_platform
```

Groups let you route alerts to the team that owns the model rather than to
whoever built the pipeline. Worth doing once a conversion has produced enough
models that "the data team" is too broad an owner.

---

Previous: [J3 · Scheduled full-refresh reconciliation](J3-scheduled-reconciliation.md) ·
Next: [J5 · Ownership and handover](J5-ownership-handover.md)
