# 23e — Customizing where models land (`generate_schema_name`)

← Previous&nbsp; [`23d-compiled-sql.md`](23d-compiled-sql.md) &nbsp;·&nbsp; Next →&nbsp; [`24-dbt-docs.md`](24-dbt-docs.md)

---

There's one dbt convention that confuses every beginner the first time they see
it in a real project: the way dbt decides which BigQuery dataset (called "schema"
in dbt-speak) your models land in.

The default behavior is surprising. Real projects override it. This file shows
both.

~15 minutes.

## The default behavior is weird

You'd think: "I set `dataset: dbt_tutorial` in `profiles.yml`. So my models land
in `dbt_tutorial`. End of story."

That's true **until** you put a `+schema: reports` config in `dbt_project.yml`
(or in a `{{ config(schema='X') }}` block). Then dbt does something surprising:

```yaml
# dbt_project.yml
models:
  my_first_project:
    marts:
      +schema: reports
```

You'd expect marts to land in `reports`. They don't. They land in
**`dbt_tutorial_reports`** — dbt **concatenates** the profile schema with the
custom schema.

This is intentional. dbt's reasoning: you might share a BigQuery project with
other developers; you don't want your dev models clobbering each other. So dbt
*always* prefixes with your profile dataset.

But in real projects, this default is wrong. You usually want:
- **In dev**: keep the prefix (so devs don't collide).
- **In prod**: no prefix — `+schema: reports` should literally mean the `reports`
  dataset.
- **In CI**: a special prefix per PR (`dbt_ci_pr_42_reports`) so PRs don't
  conflict.

You override this with a special macro called `generate_schema_name`.

## Step 1: See the default in action (optional but instructive)

Open `dbt_project.yml`. Find the `models:` block and add a `+schema` override:

```yaml
models:
  my_first_project:
    staging:
      +materialized: view
    marts:
      +materialized: table
      +schema: reports
```

Run:

```bash
dbt build --select dim_customers
```

Expected output:

```
1 of 1 OK created sql table model dbt_tutorial_reports.dim_customers
```

Not `dbt_tutorial.dim_customers`. Not `reports.dim_customers`. **`dbt_tutorial_reports.dim_customers`** — the profile dataset concatenated with `reports`.

Open BigQuery in the console. You'll see a NEW dataset called `dbt_tutorial_reports`.
Yes, dbt created it. The `dim_customers` table is inside it.

This is the dbt default at work. The macro that produced this name is built-in.

## Step 2: Look at the default macro

dbt's built-in `generate_schema_name` macro, paraphrased:

```sql
{% macro generate_schema_name(custom_schema_name, node) -%}
    {%- set default_schema = target.schema -%}
    {%- if custom_schema_name is none -%}
        {{ default_schema }}
    {%- else -%}
        {{ default_schema }}_{{ custom_schema_name | trim }}
    {%- endif -%}
{%- endmacro %}
```

Read it:

- If you didn't set a custom schema → use the profile's schema as-is
  (`dbt_tutorial`).
- If you DID set a custom schema → concatenate (`dbt_tutorial_reports`).

That's the default. To change the behavior, you write your **own**
`generate_schema_name` and dbt uses yours instead.

## Step 3: Override the macro

Create `macros/generate_schema_name.sql`:

```sql
{% macro generate_schema_name(custom_schema_name, node) -%}
    {%- set default_schema = target.schema -%}
    {%- if target.name == 'prod' -%}
        {# In prod: custom schema wins outright, no prefix. #}
        {%- if custom_schema_name is none -%}
            {{ default_schema }}
        {%- else -%}
            {{ custom_schema_name | trim }}
        {%- endif -%}
    {%- else -%}
        {# In dev/ci: keep default concatenation behavior. #}
        {%- if custom_schema_name is none -%}
            {{ default_schema }}
        {%- else -%}
            {{ default_schema }}_{{ custom_schema_name | trim }}
        {%- endif -%}
    {%- endif -%}
{%- endmacro %}
```

What this macro says:

- If `target.name == 'prod'`: a `+schema: reports` config means literally
  `reports`. No prefix.
- Otherwise (dev, ci, etc.): keep the default concatenation behavior.

## Step 4: See the override take effect

Run the dev build:

```bash
dbt build --select dim_customers
```

Expected: `dbt_tutorial_reports.dim_customers` — same as before. Dev behavior
unchanged.

Now switch to prod (you set this up in file 23c):

```bash
dbt build --target prod --select dim_customers
```

Expected output:

```
1 of 1 OK created sql table model reports.dim_customers
```

**Just `reports`. No prefix.** That's the override.

Confirm in BigQuery: a new dataset called `reports` exists, with `dim_customers`
inside. (Created on demand because dbt asks BQ to create the dataset if missing.)

## Step 5: The CI ephemeral-schema pattern (real-world)

This is what spotify-pipeline's CI does. The idea: in CI runs (`--target ci`),
*every PR gets its own isolated dataset* so PR builds don't collide with each
other or with the main dev/prod environments.

The override pattern:

```sql
{% macro generate_schema_name(custom_schema_name, node) -%}
    {%- set default_schema = target.schema -%}
    {%- if target.name == 'prod' -%}
        {%- if custom_schema_name is none -%}
            {{ default_schema }}
        {%- else -%}
            {{ custom_schema_name | trim }}
        {%- endif -%}
    {%- elif target.name == 'ci' -%}
        {# CI builds go to a per-PR dataset. DBT_CI_DATASET is set by the CI workflow. #}
        {%- set ci_dataset = env_var('DBT_CI_DATASET', default_schema) -%}
        {%- if custom_schema_name is none -%}
            {{ ci_dataset }}
        {%- else -%}
            {{ ci_dataset }}_{{ custom_schema_name | trim }}
        {%- endif -%}
    {%- else -%}
        {# dev — default dbt behavior. #}
        {%- if custom_schema_name is none -%}
            {{ default_schema }}
        {%- else -%}
            {{ default_schema }}_{{ custom_schema_name | trim }}
        {%- endif -%}
    {%- endif -%}
{%- endmacro %}
```

Then in the CI workflow (GitHub Actions), set `DBT_CI_DATASET=dbt_ci_pr_42`
before running `dbt build --target ci`. Every model builds into
`dbt_ci_pr_42.*`, separate from anyone else's CI build.

When the PR closes, a cleanup job drops the entire `dbt_ci_pr_42` dataset. Clean
slate. No leftover tables.

You can read this exact pattern in
[`spotify-pipeline`'s dbt project](https://github.com/edwinrdrr/spotify-pipeline/tree/main/dbt).

## Step 6: Clean up

Open `dbt_project.yml` and **remove** the `+schema: reports` line you added in
Step 1:

```yaml
models:
  my_first_project:
    staging:
      +materialized: view
    marts:
      +materialized: table
```

Then drop the now-orphaned datasets if you want clean BigQuery:

```bash
bq rm -r -f your-project:dbt_tutorial_reports
bq rm -r -f your-project:reports
```

(`-r` deletes contents, `-f` skips the confirmation prompt.)

Keep `macros/generate_schema_name.sql` around — it's harmless when no `+schema`
configs exist (the `is none` branch covers the default case).

## When you'd actually use this

| Situation | Why you override |
|---|---|
| Multi-environment project (dev/ci/prod) | Strip the dev-prefix convention in prod so clean dataset names |
| Per-PR CI ephemeral schemas | Isolate every PR's models from each other |
| Multi-tenant — schemas per customer | `target.name == 'tenant_a'` → tenant-specific schema |
| You want all marts in one logical dataset | Without the override, `+schema: marts` becomes `dev_marts` instead of `marts` |

If you're a solo dev on a learning project, you don't need this. The default is
fine. The moment your project has more than one developer or CI runs, you'll
reach for it.

## What did you just do?

- Saw the default `generate_schema_name` behavior (prefix concatenation).
- Wrote a custom `generate_schema_name` macro.
- Verified the override works in `prod` (no prefix) but leaves dev alone.
- Saw the CI ephemeral-schema pattern used by spotify-pipeline.

## Common confusion

> "Why is dbt's default to concatenate? It's so confusing."

Historical reason: dbt was originally used by analytics teams sharing one
Redshift cluster. The prefix was a safety net to prevent devs from clobbering
each other's models in shared schemas. It's defensible — just often wrong for
modern multi-environment setups.

> "Do I have to write this macro for every project?"

Yes — there's no built-in option, you have to write it. Every serious dbt
project has a `macros/generate_schema_name.sql`. Copy-paste from any reference
project.

> "What about `generate_database_name`?"

Same pattern, different macro. Override `generate_database_name` to control
which **project** (BigQuery term) models land in. Less commonly customized.

> "What about `generate_alias_name`?"

Controls the **table name** within a schema. Mostly the same model name as the
file, unless you override.

> "Does this affect tests?"

Tests live in the schema of the model they're testing. If your model went to
`dbt_tutorial_reports`, the test for it queries from there too.

---

You can now read any dbt project's `macros/generate_schema_name.sql` and
understand exactly what it does. That's the last piece of the "advanced
configuration" puzzle. Next: see your project documented as a website.

← Previous&nbsp; [`23d-compiled-sql.md`](23d-compiled-sql.md) &nbsp;·&nbsp; Next →&nbsp; [`24-dbt-docs.md`](24-dbt-docs.md)
