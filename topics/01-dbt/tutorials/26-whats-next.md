# 26 — What's next

← Previous&nbsp; [`25-common-errors.md`](25-common-errors.md)

---

You finished the dbt course. Here's where to go from here.

## What you can now do

After this course, you can:

- ✅ Set up a dbt project against BigQuery from scratch.
- ✅ Understand `dbt_project.yml`, `profiles.yml`, and how they connect.
- ✅ Read and write models using `{{ ref(...) }}` and `{{ source(...) }}`.
- ✅ Recognize the staging → marts layered architecture.
- ✅ Add column tests and understand what each one checks.
- ✅ Use `dbt build` for the day-to-day workflow.
- ✅ Choose between view, table, incremental, and ephemeral materializations.
- ✅ Generate and serve docs with the lineage graph.

You should now be able to read any small-to-medium dbt project's code and follow
along — **including spotify-pipeline's `dbt/` folder**, which is your next stop.

## Step 1: Read `applied-in-spotify-pipeline.md`

In this folder's parent (`topics/01-dbt/`), open:

[`../applied-in-spotify-pipeline.md`](../applied-in-spotify-pipeline.md)

That file walks through how every concept you just learned shows up in a
production-style project. Click through to the actual `dbt/` files on GitHub
and see them as full implementations.

## Step 2: Practice more

You learned by following along. Now practice by doing without a tutorial:

### Idea A: Extend the project you just built

- Add a `fct_orders` mart (one row per order, joined to customer info).
- Add a `dim_dates` mart (one row per calendar day, useful for joining).
- Add an `accepted_values` test on `dim_customers.orders_count` to verify it's
  always >= 0.
- Add a custom test in `tests/`: "no customer has signed up after their first
  order date."

### Idea B: Bring your own data

Pick a CSV (or many) from a hobby project, your GitHub activity, your
banking exports — anything. Load as seeds, build staging + marts, add tests.
The leap from "I followed a tutorial" to "I can build" happens here.

### Idea C: Convert seeds to sources

So far we used `{{ ref('raw_customers') }}` on a seed. Now try:

1. Move the data into BigQuery via `bq load` (a manual ingestion).
2. Declare it as a source in `_sources.yml`.
3. Switch `stg_customers` to use `{{ source('raw_group', 'customers') }}` instead.

This teaches you the source pattern that spotify-pipeline and most real projects
use.

## Step 3: Try Slim CI

You ran `dbt build` to rebuild everything. Real CI is smarter:

```bash
# After committing prod's manifest somewhere:
dbt build --select state:modified.body+ --defer --state ../path-to-prod-manifest/
```

`state:modified.body+` picks only models whose SQL body changed (and their
downstream). `--defer` says: for models that didn't change, use the prod version
(read from prod, don't rebuild in CI). The combo makes CI 5-10x faster on big
projects.

Read [spotify-pipeline doc 09 — Slim CI](https://github.com/edwinrdrr/spotify-pipeline/blob/main/docs/setup/09-slim-ci.md)
for how this is wired up in production.

## Step 4: Learn incremental models properly

We skipped incremental in this course because we have 10 rows. Try it next:

- Create a `fct_orders` model.
- Materialize as `incremental`.
- Add a `is_incremental()` block to only process new rows.
- Add 5 more rows to `raw_orders.csv`. Re-build. See that only those 5 land in
  the table (not a full rebuild).

[dbt's incremental materializations doc](https://docs.getdbt.com/docs/build/incremental-models)
is the canonical reference.

## Step 5: Pick up the dbt ecosystem

Three things to dip into when you're comfortable with the basics:

### Packages

dbt has a package ecosystem (like npm/pip). Most projects use
[`dbt-utils`](https://hub.getdbt.com/dbt-labs/dbt_utils/latest/) for things like
`generate_surrogate_key()` and `date_spine()`. Try installing one and using a
helper.

### Macros

When you find yourself writing the same Jinja-flavored SQL in multiple models,
factor it into a macro in `macros/`. Macros are dbt's reusable code.

### `dbt-bigquery`-specific features

BigQuery has features (partitioned tables, clustered tables) that dbt exposes
via config:

```sql
{{ config(
    materialized='table',
    partition_by={'field': 'order_date', 'data_type': 'date'},
    cluster_by=['customer_id']
) }}
```

Partitioning + clustering is how real projects make big-table queries fast and
cheap.

## Step 6: Read other people's dbt code

The best way to internalize patterns is to read real projects:

- **spotify-pipeline** — the obvious one: small enough to read end-to-end, full
  of real-world patterns (Slim CI, env-aware profiles, source declarations).
- **[dbt Labs's `jaffle_shop_duckdb`](https://github.com/dbt-labs/jaffle_shop_duckdb)**
  — the canonical demo. Slightly larger than what you built.
- **[GitLab's dbt repo](https://gitlab.com/gitlab-data/analytics)** — huge,
  enterprise-scale, public. Reading it is intimidating but instructive.

## Step 7: Move to the next topic

When you're ready, the de-learning-notes repo has more topics queued up:

- **02-terraform** — already has `concepts.md`, `cheatsheet.md`, and `applied-in-spotify-pipeline.md`. Tutorials coming.
- **03-github-actions**, **04-iam**, **05-wif** — stubs to be filled.

Pick whichever feels relevant. For continuing spotify-pipeline understanding,
Terraform is the natural next step (you've learned the transform layer; now learn
how the underlying infrastructure is managed).

## Final thoughts

You came in not knowing what `profiles.yml` was for. You came out knowing how to
build a dbt project from scratch, run tests on it, and make a documented
website explaining what you did.

The next leap — from "can use dbt" to "use it productively" — comes from typing
SQL on real datasets. Not more reading. Build something.

Good luck.

← Previous&nbsp; [`25-common-errors.md`](25-common-errors.md)
