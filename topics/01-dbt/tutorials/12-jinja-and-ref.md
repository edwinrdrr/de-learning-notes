# 12 — Jinja, `ref()`, and `source()`

← Previous&nbsp; [`11-seeds.md`](11-seeds.md) &nbsp;·&nbsp; Next →&nbsp; [`13-first-staging-model.md`](13-first-staging-model.md)

---

You've seen `{{ ref(...) }}` and `{{ config(...) }}` in the example models. Time to
slow down and understand what's actually happening when you write those.

## What is Jinja?

**Jinja** is a Python templating engine. It lets you embed expressions and logic
inside text. dbt uses Jinja inside `.sql` files (and `.yml` files) to add features
SQL doesn't have on its own: variables, conditionals, references to other models,
function calls.

The full Jinja syntax has three building blocks:

| Syntax | What it does |
|---|---|
| `{{ ... }}` | **Expression**: evaluates and substitutes the result. |
| `{% ... %}` | **Statement**: runs logic (loops, conditionals) but doesn't output. |
| `{# ... #}` | **Comment**: removed at render time. |

In dbt, you'll see `{{ ... }}` constantly, `{% ... %}` sometimes (in macros), and
`{# ... #}` rarely.

## Compile-time vs run-time

This is the key insight: **Jinja runs on your laptop, not in BigQuery.**

When you run `dbt run`, dbt:

1. Reads `my_model.sql`.
2. Renders the Jinja: every `{{ ... }}` is evaluated and the result substituted
   in. This produces a plain SQL string.
3. Sends that plain SQL string to BigQuery to execute.

BigQuery never sees Jinja. It only sees the rendered SQL.

You can confirm by running `dbt compile` (no warehouse hits) and then looking at
`target/compiled/...`. You'll find your `.sql` files with all the `{{ ... }}` parts
replaced by their resolved values.

## `{{ ref('model_name') }}` — the most important one

`ref()` is a Jinja function dbt provides. You pass it the name of another model (or
seed), and it returns the **fully-qualified table name** of that thing.

Example. Suppose you write:

```sql
select * from {{ ref('raw_customers') }}
```

dbt's compile step finds the `raw_customers` thing (a seed in our case), looks up
its actual location (`<your-project>.dbt_tutorial.raw_customers`), and substitutes:

```sql
select * from `sweet-mountain-471812`.`dbt_tutorial`.`raw_customers`
```

That rendered SQL is what BigQuery executes.

### Why this matters

Three reasons `ref()` exists instead of just hardcoding `<project>.dbt_tutorial.raw_customers`:

1. **Environment portability.** When you switch from `dev` to `prod`, the project
   and dataset change. `ref()` resolves to the right values per target — your SQL
   doesn't change.
2. **DAG construction.** dbt scans every model for `ref()` calls. From that, it
   builds the dependency graph. `dbt run --select foo+` (foo and everything that
   depends on it) is possible because dbt knows the dependencies.
3. **Build order.** dbt knows that if `model_B` refs `model_A`, then `model_A` must
   be built first. You never tell dbt the order — `ref()` is how it figures it out.

### What `ref()` can point at

`ref('name')` can point at:
- Another model (a `.sql` file in `models/`).
- A seed (a `.csv` file in `seeds/`).
- A snapshot (in `snapshots/`).

All three end up as tables in the warehouse; `ref()` finds them by name regardless
of which kind they are.

## `{{ source('source_name', 'table_name') }}` — for raw tables

`ref()` only finds things dbt itself built. But what about raw tables — ones loaded
by an ingestion pipeline, sitting in the warehouse already, not built by dbt?

For those, you use **`source()`**.

`source()` looks up a table you've declared in a `_sources.yml` file. Example:

```yaml
# models/staging/_sources.yml
version: 2
sources:
  - name: raw_data
    database: my-project-id        # = BigQuery project
    schema: raw                     # = BigQuery dataset
    tables:
      - name: orders
```

Then in a model:

```sql
select * from {{ source('raw_data', 'orders') }}
```

dbt compiles to:

```sql
select * from `my-project-id`.`raw`.`orders`
```

### Why `source()` vs hardcoding the table name?

Same reasons as `ref()`:

1. **Environment portability**: the `database:` and `schema:` can use env vars.
2. **DAG visibility**: dbt knows your models depend on these external tables.
3. **Tests on sources**: you can declare `not_null`/`unique` tests on source tables
   (test the raw data before transforming).
4. **Source freshness**: dbt can check "was this source table updated in the last
   N hours?" — useful for spotting broken ingestion.

### When to use which

- `{{ ref('name') }}` → for anything dbt built (other models, seeds, snapshots).
- `{{ source('src', 'table') }}` → for tables that already exist in your warehouse
  before dbt runs (raw ingestion landing tables).

In **this tutorial**, we'll use `ref()` to point at our seeds. Real projects use
`source()` for production-level raw data. spotify-pipeline uses `source()` —
look at
[`dbt/models/staging/_spotify__sources.yml`](https://github.com/edwinrdrr/spotify-pipeline/blob/main/dbt/models/staging/_spotify__sources.yml).

## `{{ config(...) }}` — model-level overrides

You've seen this in the example model:

```sql
{{ config(materialized='table') }}
```

`config()` is a Jinja call that sets configuration for *this specific model*.
Common options:

```sql
{{ config(
    materialized='table',
    tags=['nightly'],
    schema='reports'
) }}
```

Project-level defaults (in `dbt_project.yml`) apply unless overridden by a
`config()` call in the model itself.

## Other Jinja you'll see (briefly)

You won't need these yet, but for context:

- **`{{ var('my_var') }}`** — read a variable defined in `dbt_project.yml` or
  passed via `--vars`.
- **`{{ env_var('MY_ENV') }}`** — read an environment variable (used in
  `profiles.yml` to keep secrets out of files).
- **`{{ this }}`** — the current model. Useful in incremental models.
- **`{% if is_incremental() %}` ... `{% endif %}`** — Jinja statements
  for conditional SQL. Used in incremental models to only process new rows on
  subsequent runs.
- **Macros** — your own custom Jinja functions, defined in `macros/`. Used to DRY
  up repeated SQL patterns.

## Try this

Let's see Jinja in action without writing a model yet. Make a temporary file:

```bash
cat > models/check_jinja.sql <<'EOF'
-- This is plain SQL with one Jinja substitution

select
    'hello' as greeting,
    '{{ target.project }}' as compiled_project_id,
    '{{ target.dataset }}' as compiled_dataset
EOF
```

Now compile it (no warehouse call):

```bash
dbt compile --select check_jinja
```

Then look at what dbt rendered:

```bash
cat target/compiled/my_first_project/models/check_jinja.sql
```

You'll see the actual values replaced into the SQL:

```sql
-- This is plain SQL with one Jinja substitution

select
    'hello' as greeting,
    'sweet-mountain-471812' as compiled_project_id,
    'dbt_tutorial' as compiled_dataset
```

That's Jinja: variables from `profiles.yml` ended up baked into the SQL at compile
time.

Delete the throwaway file:

```bash
rm models/check_jinja.sql
```

## What did you just do?

- Learned what Jinja is and the three syntax forms.
- Understood that Jinja runs on your laptop, not in the warehouse — compile
  happens before any SQL is sent.
- Got the difference between `ref()` (dbt-built things) and `source()` (raw
  tables loaded by other means).
- Saw Jinja substitution happen by running `dbt compile` and reading the rendered
  SQL.

## Common confusion

> "Why does dbt use Jinja and not plain SQL?"

Because plain SQL doesn't have variables, conditionals, or function calls. You
can't write "if it's the first run, do X, otherwise Y" in pure SQL. Jinja gives
SQL a templating layer. The downside: another thing to learn. The upside: it
unlocks features like ref/source/incremental that would otherwise require external
tooling.

> "Can I write a SQL model with no Jinja at all?"

Yes. A pure-SQL model is valid:

```sql
select 1 as x
```

But you'd have hardcoded table names everywhere, no DAG awareness, no environment
portability. Almost every real model has at least one `{{ ref(...) }}`.

> "What if I don't recognize a `{{ ... }}` in someone's code?"

`dbt compile` is your friend. Compile, then read the rendered SQL in
`target/compiled/`. Whatever the Jinja resolved to is right there.

---

You understand the substitution. Now let's write a real model.

← Previous&nbsp; [`11-seeds.md`](11-seeds.md) &nbsp;·&nbsp; Next →&nbsp; [`13-first-staging-model.md`](13-first-staging-model.md)
