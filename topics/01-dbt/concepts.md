# dbt — concepts

## What is it?

**dbt** (data build tool) is a command-line tool that lets you write your data
transformations as **SELECT statements in SQL files** instead of stitched-together
Python scripts or Airflow tasks. You write what each table should look like; dbt
figures out the order to build them, runs them against your warehouse, and runs tests
on the results.

It does NOT move data. It only transforms data that's *already* in your warehouse
(BigQuery, Snowflake, Redshift, etc.). The "E" and "L" of ELT happen elsewhere; dbt
is the "T."

## What problem does it solve?

Before dbt, transformation was a mess:
- Each query lived in a different place (a Looker file, a stored procedure, a Python
  script, somebody's notebook).
- When `dim_customers` changed, you had no idea which dashboards broke.
- "Did this query already run today?" was an honest question with no easy answer.
- Tests on data ("this column should never be null") were rare because they had no
  good home.

dbt solves all four by making transformation **a project**, like software:
- Every model is a `.sql` file in a folder, version-controlled.
- References between models are explicit (`{{ ref('foo') }}`), so dbt builds a DAG.
- A single command (`dbt build`) compiles, runs, and tests in the right order.
- Tests are first-class objects declared next to the models they protect.

## The mental model

Think of dbt as **a compiler for SQL**.

You write:
```sql
-- models/staging/stg_users.sql
select id, email from {{ source('raw', 'users') }}
```

dbt compiles to (literally):
```sql
create or replace view my_project.staging.stg_users as
select id, email from raw_project.raw.users
```

The `{{ ref(...) }}` and `{{ source(...) }}` macros are how dbt knows what depends on
what. From those references, dbt builds a graph of all your models and runs them in
dependency order.

### Layered architecture (the common pattern)

```
sources         (raw landing tables — owned by ingestion, not dbt)
   ↓
staging         (one model per source — light cleanup, renames, casts)
   ↓
marts           (analytics-ready facts and dimensions)
```

The convention: never `ref()` directly to a source from a mart. Always go
`source → staging → mart`. Keeps blast radius small when an ingestion column renames.

## Key terms

| Term | Meaning |
|---|---|
| **model** | A `.sql` file in `models/`. Becomes a table or view in the warehouse. |
| **source** | A raw table that already exists in the warehouse (not built by dbt). Declared in a `sources.yml`. |
| **ref** | `{{ ref('other_model') }}` — references another dbt model by name. Builds the DAG. |
| **materialization** | How the model is built: `table` (rebuild every run), `view` (lazy), `incremental` (append/merge new rows), `ephemeral` (inlined as a CTE). |
| **test** | A SQL query that should return zero rows if data is healthy. Declared in `.yml` (built-in: `not_null`, `unique`, `accepted_values`, `relationships`) or written from scratch (custom tests). |
| **target** | An environment in `profiles.yml` (e.g., `dev`, `prod`). `dbt build --target prod` builds against the prod connection. |
| **seed** | A CSV file in `seeds/` that dbt loads as a table. For tiny static reference data, not real data. |
| **snapshot** | dbt's slowly-changing-dimension (SCD) tooling — capture how a row looked over time. |
| **state:modified.body+** | A selector that picks models whose SQL body changed since a reference manifest, plus everything downstream. Used in Slim CI. |
| **--defer** | Tells dbt: for models not in this build, read them from a different environment (usually prod). Halves CI build time. |

## When not to use dbt

- **Real-time / streaming**: dbt is batch (run-on-a-schedule). It's not for sub-second
  latency requirements.
- **The L of ELT**: dbt doesn't load raw data into your warehouse. Use ingestion tools
  (Fivetran, Airbyte, custom Python) for that.
- **Tiny projects** (one analyst, one dashboard, three tables): the dbt setup cost
  outweighs the benefit. A single SQL view will do.

## Common misuses

- **Putting business logic in views.** Materialize as `table` if downstream is slow.
- **Skipping `staging/`** — going straight from `source` to `mart`. Breaks when the
  source schema changes; you'll edit every mart.
- **Tests that don't fail when data is wrong.** A test is only useful if it actually
  catches a bug. Run `dbt test` against known-bad data once to verify.
- **One giant mart**. Split by grain (one customer-day fact, one customer dim).
- **Manual `dbt run` on prod**. Once you have CI, that's the only path to prod.

## Recommended external reading

- [dbt Developer Docs](https://docs.getdbt.com/docs/introduction) — the official docs
  are excellent; read the "Build a project" tutorial.
- [Best practices guide](https://docs.getdbt.com/best-practices) — the layered
  architecture (`staging` / `intermediate` / `marts`) is documented in detail.
- [SQL Style Guide](https://docs.getdbt.com/best-practices/how-we-style/0-how-we-style-our-dbt-projects) —
  pick a style and stick to it; this is dbt Labs's own.
