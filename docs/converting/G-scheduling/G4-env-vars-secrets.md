# G4 · Environment variables and secrets

> **Part G — Scheduling, parameters, backfills** · Sourcing: `CORE`
> **The question:** my script reads credentials and config from the environment. Where does that go?

`env_var()` for config. For credentials, mostly nowhere — dbt authenticates
through the profile, and a converted model rarely needs a secret at all.

## `env_var` in the profile

```yaml
my_project:
  outputs:
    prod:
      type: bigquery
      method: service-account
      project: "{{ env_var('GCP_PROJECT') }}"
      keyfile: "{{ env_var('GCP_KEYFILE_PATH') }}"
      dataset: analytics
      threads: 8
```

And for non-secret config:

```yaml
vars:
  export_bucket: "{{ env_var('EXPORT_BUCKET', 'acme-dev-exports') }}"
```

Give a default for anything non-secret. Don't give one for a secret — you want a
missing credential to fail immediately, not to silently fall back.

## Most scripts don't need secrets after conversion

Scripts held credentials because they authenticated themselves. A dbt model
doesn't — the profile handles auth, and the model is just SQL.

So when you find credential handling in a script, the usual answer is that it
disappears:

| Script does | After conversion |
| --- | --- |
| Loads a service-account key to query BigQuery | The profile — gone from the model |
| Reads a webhook URL to notify | Orchestrator — [D13](../D-data-movement/D13-notifications.md) |
| Reads an API key to fetch data | Not a model — ingestion stays outside |
| Reads a GCS path to export to | A var, if the path varies by environment |

Only the last is a model concern, and it isn't secret.

## Prefer workload identity over keyfiles

If the script used a downloaded key file, conversion is a good moment to stop.
Workload identity, or the ambient service account on the compute running dbt,
removes the key entirely:

```yaml
my_project:
  outputs:
    prod:
      type: bigquery
      method: oauth          # ambient credentials
      project: "{{ env_var('GCP_PROJECT') }}"
```

Fewer secrets to rotate, and it's usually less work than porting the key handling.

## Never put secrets in vars on the command line

```bash
# don't
dbt run --vars '{api_key: sk-abc123}'
```

That lands in shell history, process listings, and CI logs. Use `env_var` in the
profile, sourced from your secret manager.

Same for `dbt_project.yml` — it's version-controlled, so anything in it is
published to everyone with repo access.

## The permissions change nobody plans for

The script ran as whatever principal the scheduler used — often a person's
account, or a service account with broad grants accumulated over years.

dbt runs as the dbt service account, which is usually newer and narrower. Check
before cutover, not after:

```sql
select user_email, count(*) as jobs
from `region-eu`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
where creation_time > timestamp_sub(current_timestamp(), interval 30 day)
  and destination_table.table_id = 'daily_events'
group by 1;
```

If that returns a principal that isn't your dbt service account, you have a
permissions task in front of the conversion. This is the single most common
"worked in dev, failed in prod" cause, alongside
[cross-project access](../E-translation/E4-cross-project-references.md).

## Separate targets, separate credentials

```yaml
my_project:
  target: dev
  outputs:
    dev:
      dataset: dbt_alice
      project: "{{ env_var('GCP_PROJECT_DEV') }}"
    prod:
      dataset: analytics
      project: "{{ env_var('GCP_PROJECT_PROD') }}"
```

Scripts typically had one hardcoded environment, which is why dev runs against
prod is such a common legacy. Fixing it is a genuine improvement, and it's free
at conversion time.

---

Previous: [G3 · Passing dates in](G3-passing-dates.md) ·
Next: [G5 · Backfill via `--full-refresh`](G5-backfill-full-refresh.md)
