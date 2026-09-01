# D13 · Notification side-effects

> **Part D — Data movement, DDL, metadata** · Sourcing: `CRAFT`
> **The question:** my script posts to Slack when it finishes. Where does that go?

Your orchestrator. dbt reports outcomes; it doesn't deliver them, and building
delivery into models puts it in the one place that can't report a failure to run.

## The pattern

```sql
-- at the end of the script
CALL ops.notify_slack('daily batch complete');
```

Or in the shell wrapper:

```bash
bq query < daily.sql || curl -X POST "$SLACK_WEBHOOK" -d '{"text":"failed"}'
```

## Why not a hook

A post-hook notification has a specific and disqualifying flaw: **it doesn't run
when the model fails** ([F16](../F-hooks/F16-hooks-and-failure.md)).

So you get notified on success and silent on failure, which is the opposite of
what a notification is for.

`on-run-end` does run after failed nodes, so it's the better dbt-side option —
but it's still the wrong layer. Notification needs to fire when dbt itself fails
to start, times out, or the container dies, and nothing inside dbt can do that.

## Where it goes

**The orchestrator.** Airflow callbacks, Cloud Composer alerting, a shell wrapper,
GitHub Actions notifications. Whatever runs dbt already knows whether it
succeeded, and it survives dbt not running at all.

```python
dbt_build = BashOperator(
    task_id='dbt_build',
    bash_command='dbt build --select +daily_summary',
    on_failure_callback=notify_slack,
)
```

**Or `on-run-end`, for run *content* rather than run status.** A summary of what
was built, row counts, which tests warned — that's about the run's results, and
`results` is available there ([F14](../F-hooks/F14-on-run-start-end.md)).

The split: **status ⇒ orchestrator. Content ⇒ `on-run-end`.**

## What dbt gives you to notify with

Rather than instrumenting models, consume the artefacts:

- **`run_results.json`** — per-node status, timing, `rows_affected`
- **Exit code** — non-zero on failure, which is what your orchestrator checks
- **`--warn-error`** — turns warnings into failures, if warnings should page

A notification step that reads `run_results.json` gives a far richer message than
a hook ever could, and it works whether the run failed at node 3 or never started.

## Alerting on data, not on runs

If the script's notification was really about **data** — "no rows arrived today" —
that isn't a notification, it's a test:

```sql
-- tests/daily_events_has_today.sql
select 1
from (select count(*) as n from {{ ref('daily_events') }}
      where event_date = current_date())
where n = 0
```

Then the test failure flows through the same alerting path as everything else. One
mechanism instead of two, and it blocks downstream models rather than just
mentioning the problem — [D12](D12-assert-gates.md).

That's usually the right conversion: most script notifications are data
assertions wearing a Slack webhook.

## Don't call webhooks from SQL

If you're tempted to use a remote function or an external UDF to post from inside
a model: don't. You get a network call inside a query, retried on query retry,
running in dev, with no visibility. The failure modes are bad and the debugging is
worse.

---

Previous: [D12 · `ASSERT` data-quality gates](D12-assert-gates.md) ·
Next: [D14 · Audit and metadata table writes](D14-audit-writes.md)
