# G10 · State-based selection and slim CI

> **Part G — Scheduling, parameters, backfills** · Sourcing: `CORE✓`
> **The question:** can CI build only what changed?

Yes — `state:modified` compares against a previous manifest. It's the thing that
makes CI viable on a project full of converted models, because rebuilding all of
them on every pull request isn't.

## The mechanism

dbt-core exposes `state` as a selection method (`StateSelectorMethod`), comparing
the current project against a manifest from a previous run:

```bash
dbt build --select state:modified+ --state ./prod-artifacts
```

`state:modified` is everything that changed; the `+` adds descendants, because a
changed model can break what reads it.

You need `manifest.json` from a known-good run — usually downloaded from your last
production build in CI.

## Why it matters after a conversion

A conversion adds a lot of models at once. Without state selection, every pull
request rebuilds all of them, and CI becomes slow enough that people stop waiting
for it.

With it, a PR touching one model builds that model and its descendants. Minutes
instead of hours.

## Defer, so you don't rebuild upstream

```bash
dbt build --select state:modified+ --defer --state ./prod-artifacts
```

`--defer` makes unselected `ref()`s resolve to the **production** relation instead
of building a dev copy. So a change to a mart reads production staging models
rather than rebuilding them.

That's the difference between a CI run costing pennies and costing what a full
build costs.

## The catch for incremental models

In a CI schema, an incremental model has no existing table — so `is_incremental()`
is false and it does a **full build**. Your carefully bounded incremental model
scans everything.

For a converted model over years of history that can be expensive enough to matter.
Mitigations:

**Limit data in CI:**

```sql
{% if target.name == 'ci' %}
  where event_date >= date_sub(current_date(), interval 3 day)
{% endif %}
```

**Exclude the expensive ones**, accepting weaker coverage:

```bash
dbt build --select state:modified+ --exclude tag:expensive
```

**Build them as views in CI**, so nothing materialises.

The first is usually right, and it's worth adding as part of the conversion rather
than after CI times out.

## What counts as modified

Changes to the model's SQL, its config, its schema YAML, and its upstream
dependencies. Note config changes count — adding a tag marks a model modified,
which can select more than you expect.

`state:new` selects only nodes absent from the previous manifest, which during a
staged conversion is a useful way to run just what you've added.

## Not just for CI

State selection is useful in production too:

```bash
# rerun what failed last time, and anything downstream
dbt build --select result:error+ --state ./last-run
```

That's the equivalent of rerunning a failed script — except it also rebuilds what
depended on it, which the script never did. `dbt retry` does the same thing more
directly.

## Keep the artefacts

State selection needs `manifest.json` from a trusted run. Publish it as a build
artefact from your production job and download it in CI.

If the manifest is stale or missing, `state:modified` silently selects more than
you want — often everything. Worth failing the CI job explicitly when the
artefact isn't found, rather than letting it fall back to a full build.

---

Previous: [G9 · Selectors](G9-selectors.md) ·
Next: [G11 · Retry and partial-failure semantics](G11-retry-and-failure.md)
