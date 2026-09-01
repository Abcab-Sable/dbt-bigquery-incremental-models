# I5 · Telling downstream consumers

> **Part I — Migration strategy** · Sourcing: `CRAFT`
> **The question:** who needs to know, and what do I tell them?

Everyone who reads the table, and specifically the differences you already
predicted in [H11](../H-verification/H11-differences-that-should-exist.md).

## Find them

The dbt DAG shows dbt consumers. It won't show the dashboards, notebooks,
spreadsheets and other teams' scripts, which is where the surprises live.

```sql
select
    user_email,
    count(*) as queries,
    max(creation_time) as last_seen
from `region-eu`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
where creation_time > timestamp_sub(current_timestamp(), interval 90 day)
  and query like '%daily_events%'
group by 1
order by queries desc;
```

Ninety days, because monthly and quarterly consumers exist and are exactly the
ones who won't be watching on cutover day.

Then check BI tooling separately — Looker, Tableau and the rest often query
through a single service account, so they appear as one principal hiding many
consumers.

## What to tell them

Not "we're migrating to dbt" — nobody cares about your tooling. Tell them what
changes for them:

```
Subject: analytics.daily_events — changes from 9 September

The table keeps its name, location and schedule. Three changes:

1. `status` will be NULL where it was previously an empty string.
   If you filter `status != 'cancelled'`, those rows will now be
   EXCLUDED. Use `coalesce(status,'') != 'cancelled'` to keep current
   behaviour.

2. Days with no activity will be empty rather than retaining stale rows.
   Affects 2026-08-14 and 2026-08-21 historically.

3. Row counts for Mar–Jun 2023 increase by ~40/day. A 2023 deduplication
   step is no longer needed; the extra rows are legitimate.

Nothing else changes. Rollback available for two weeks.
Questions: #data-platform, or me.
```

Concrete, specific, with the fix for the one that bites. That's from the predicted
differences — if you can't write this email, you aren't ready to cut over.

## Record them as exposures

Once found, put them in the project so the next person doesn't have to repeat the
archaeology:

```yaml
exposures:
  - name: finance_monthly_close
    type: dashboard
    url: https://looker.acme.com/dashboards/412
    description: Monthly close. Reads daily_events; sensitive to row counts.
    depends_on: [ref('daily_events')]
    owner: {name: Priya Raman, email: priya@acme.com}
```

Now `dbt build --select daily_events+` shows the dashboard in the lineage, and
anyone changing the model sees who it affects.

## Timing

**Before** the shadow period ends — people need time to check their own things.

Not on cutover day. And not so far ahead that everyone forgets; a week's notice
with a reminder the day before works.

For the monthly and quarterly consumers, tell them **which run** is their first
one on the new model. They won't be watching daily.

## Get the impactful ones agreed

For differences with real consequences, a named person should accept them in
writing before cutover:

| Difference | Needs |
| --- | --- |
| Row counts change materially | Whoever reports the numbers |
| Null semantics change | Anyone filtering that column |
| A period becomes empty | Whoever expected data there |
| Types change | Anyone joining or casting |

Record the name and date in the sign-off ([H13](../H-verification/H13-sign-off.md)).
"This was agreed with Priya on 2 September" is what turns an incident into a
planned change.

## The one nobody tells

Someone will be missed — a quarterly report, a notebook on a laptop. Reduce the
damage by keeping rollback available for a full reporting cycle
([I6](I6-rollback-keeping-script.md)), and by making the announcement findable
after the fact rather than only in a Slack message that scrolls away.

---

Previous: [I4 · Dual-write during cutover](I4-dual-write.md) ·
Next: [I6 · Rollback: keeping the old script runnable](I6-rollback-keeping-script.md)
