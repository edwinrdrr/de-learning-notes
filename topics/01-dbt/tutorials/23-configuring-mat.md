# 23 — Configuring materializations (where to set them)

← Previous&nbsp; [`22-materializations.md`](22-materializations.md) &nbsp;·&nbsp; Next →&nbsp; [`23b-packages.md`](23b-packages.md)

---

You've seen `{{ config(materialized='table') }}` at the top of a model. You've
seen a `models:` block in `dbt_project.yml`. dbt supports setting materialization
at **three levels**, from broad to narrow. Most projects use a mix.

Short file. ~10 minutes.

## The three levels (most general → most specific)

1. **Project default**, in `dbt_project.yml`.
2. **Folder-level**, in `dbt_project.yml` scoped to a subfolder.
3. **Model-level**, with `{{ config(...) }}` at the top of the `.sql` file.

More specific levels override more general ones. So a model-level config wins
over a folder-level config, which wins over the project default.

## Level 1: project default

Open `dbt_project.yml`. The bottom has a `models:` block:

```yaml
models:
  my_first_project:
```

Right now it's empty (you cleaned it up in file 09). The implicit default for any
model is `view`.

You could set a project-wide default:

```yaml
models:
  my_first_project:
    +materialized: view
```

That says: "Every model in this project defaults to view." (Same as the implicit
default, just written out.)

Notice the `+` prefix. dbt uses `+` to mark **configurations** in
`dbt_project.yml`, to distinguish them from folder names. Without the `+`, dbt
would think you meant a subfolder called `materialized`. With the `+`, dbt knows
you mean "this is a config that applies to everything below."

## Level 2: folder-level

A more useful pattern. Set different defaults for different folders:

```yaml
models:
  my_first_project:
    staging:
      +materialized: view
    marts:
      +materialized: table
```

Reading this:
- "All models in `my_first_project`..."
- "...specifically in the `staging/` subfolder, default to view."
- "...specifically in the `marts/` subfolder, default to table."

The structure mirrors your folder layout. If you had `models/marts/finance/`,
you could set a more specific config:

```yaml
models:
  my_first_project:
    staging:
      +materialized: view
    marts:
      +materialized: table
      finance:
        +materialized: incremental    # finance marts are incremental, others are tables
```

Closer to the model, more specific.

## Level 3: model-level

```sql
{{ config(
    materialized='table',
    tags=['nightly'],
    schema='reports'
) }}

select ... from ...
```

A single model overrides everything above it. Most models don't need a config
block — they inherit folder defaults — but when you need to deviate, this is
where.

## Step 1: Set up project defaults

Open `dbt_project.yml` and edit the bottom `models:` block to:

```yaml
models:
  my_first_project:
    staging:
      +materialized: view
    marts:
      +materialized: table
```

Save.

## Step 2: Remove the redundant `{{ config(...) }}` from `dim_customers`

Now that `models/marts/+materialized: table` is the folder default, the
`{{ config(materialized='table') }}` at the top of `dim_customers.sql` is
redundant.

Open `models/marts/dim_customers.sql` and **delete** the first two lines:

```sql
{{ config(materialized='table') }}

with customers as (
```

So the file starts directly with:

```sql
with customers as (
```

Save.

## Step 3: Verify

```bash
dbt build
```

Expected: `dim_customers` still builds as a TABLE (same as before), inheriting
the folder default. No change in behavior, less boilerplate in the file.

Confirm in BigQuery: `dim_customers` is still a table, with the same data.

## When to use each level

| Use level | When |
|---|---|
| Project default | Consistent across the whole project. Rarely needed; folder-level is usually clearer. |
| Folder-level | "Every staging model is a view; every mart is a table." The standard pattern. |
| Model-level | Exceptions. One specific model needs a different materialization. |

A typical real project uses **folder-level for the common case** and **model-level
for exceptions**.

## Other configs you can set the same way

`materialized` isn't the only thing. Other common configs:

- **`schema`** — write into a different BigQuery dataset than the default. Useful
  for "all marts go to a `reports` dataset, all staging to a `staging` dataset."
- **`tags`** — label models. Then run subsets with `--select tag:nightly`.
- **`alias`** — change the table name to something different from the model file
  name.

All of them can be set at any of the three levels. The pattern is the same: `+key:
value` in `dbt_project.yml` (folder level) or `key=value` in `{{ config(...) }}`
(model level).

## What did you just do?

- Set materialization defaults at the folder level in `dbt_project.yml`.
- Removed the now-redundant model-level config from `dim_customers.sql`.
- Verified the model still builds as a table.

## Common confusion

> "Why the `+` prefix in `dbt_project.yml` but not in `{{ config(...) }}`?"

`dbt_project.yml`'s `+` prefix exists because YAML has no other way to distinguish
"this is config" from "this is a subfolder name." `models.my_project.marts:` is
ambiguous — is `marts` a config key or a folder? With `+materialized:` you know
it's a config. Inside `{{ config(...) }}` it's unambiguous (no folder names), so
no prefix needed.

> "If I set both a folder-level and a model-level config, which wins?"

Model wins. dbt resolves the most-specific config it finds.

> "Can I set the same config in multiple places without conflict?"

Yes. They're additive for non-conflicting keys, and the most-specific wins for
conflicting keys.

> "Should I always use folder-level configs and avoid `{{ config(...) }}`?"

Generally yes — it keeps the SQL files focused on SQL and the configuration in
one place. Use `{{ config(...) }}` only for genuine exceptions.

---

Materializations sorted. Last bit: see your project as a website.

← Previous&nbsp; [`22-materializations.md`](22-materializations.md) &nbsp;·&nbsp; Next →&nbsp; [`23b-packages.md`](23b-packages.md)
