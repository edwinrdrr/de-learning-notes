# 09 — Read the example models, then delete them

← Previous&nbsp; [`08-dbt-debug.md`](08-dbt-debug.md) &nbsp;·&nbsp; Next →&nbsp; [`10-dbt-run.md`](10-dbt-run.md)

---

`dbt init` scaffolded two example models for you in `models/example/`. Let's read
them, run them once to see them work, then delete them so we have a clean slate.

## Step 1: Look at what's there

From inside `my_first_project/`:

```bash
ls models/example/
```

You'll see:

```
my_first_dbt_model.sql
my_second_dbt_model.sql
schema.yml
```

Two `.sql` files (= two models) and one `.yml` file (= tests + descriptions for
those models).

## Step 2: Read the first model

```bash
cat models/example/my_first_dbt_model.sql
```

You'll see SQL roughly like this (with comments):

```sql
/*
    Welcome to your first dbt model!
    Did you know that you can use comments to add details to your queries?
*/

{{ config(materialized='table') }}

with source_data as (

    select 1 as id
    union all
    select null as id

)

select *
from source_data

/*
    Uncomment the line below to remove records with null `id` values
*/

-- where id is not null
```

Let's read it line by line.

### `{{ config(materialized='table') }}`

This is a **Jinja expression** (we'll cover Jinja in file 12). The `{{ ... }}`
syntax tells dbt "evaluate this and substitute the result." `config(materialized='table')`
tells dbt: "when building this model, materialize it as a TABLE in BigQuery, not a
view."

Without this line, the model would use whatever default is in `dbt_project.yml`.
The example overrides at the model level just to demo the syntax.

### The CTE (`with source_data as (...)`)

This is plain SQL. A Common Table Expression (CTE) is a named subquery you can
reference later. Here, `source_data` is a fake 2-row table built inline:

```sql
select 1 as id
union all
select null as id
```

That produces:
```
| id   |
|------|
| 1    |
| null |
```

Then `select * from source_data` returns those two rows.

### What this model represents

It's a model that produces a 2-row table. One row has `id = 1`, the other has
`id = null`. The whole purpose is to demonstrate that:

1. You can write SQL in a `.sql` file.
2. dbt compiles it and creates the table in BigQuery.
3. The result is a queryable table named after the file (`my_first_dbt_model`).

The commented-out `-- where id is not null` line is a hint: if you uncommented it,
the model would filter out the null row. You'd have a 1-row table.

## Step 3: Read the second model

```bash
cat models/example/my_second_dbt_model.sql
```

You'll see something like:

```sql
-- Use the `ref` function to select from other models

select *
from {{ ref('my_first_dbt_model') }}
where id = 1
```

This one is more interesting because it introduces `ref()`. The `{{ ref('my_first_dbt_model') }}`
syntax means "fill in the fully-qualified table name of the model called
`my_first_dbt_model`." dbt will resolve it at compile time to something like
`sweet-mountain-471812.dbt_tutorial.my_first_dbt_model`.

So this second model:
1. Selects from the table the first model built.
2. Filters down to `where id = 1`.
3. Results in a 1-row table (the row that survived).

Note also there's no `{{ config(...) }}` at the top of this one — so it'll use the
project default, which for the example folder is `view` (you'll see this in
`dbt_project.yml`).

## Step 4: Look at `schema.yml`

```bash
cat models/example/schema.yml
```

You'll see:

```yaml
version: 2

models:
  - name: my_first_dbt_model
    description: "A starter dbt model"
    columns:
      - name: id
        description: "The primary key for this table"
        data_tests:
          - unique
          - not_null

  - name: my_second_dbt_model
    description: "A starter dbt model"
    columns:
      - name: id
        description: "The primary key for this table"
        data_tests:
          - unique
          - not_null
```

This `schema.yml` file does two things:

1. **Documentation**: each model and column has a `description`. This shows up
   in `dbt docs` (we'll see in file 24).
2. **Tests**: `data_tests: [unique, not_null]` on the `id` column says "run a
   uniqueness check and a not-null check on this column." (Older dbt called this
   `tests:`; newer versions support both. If you see `data_tests:`, that's the
   current preferred name.)

The first model has a null `id` so `not_null` will fail on it. The example does
this intentionally to demo what a failing test looks like (we'll see in file 20).

## Step 5: Delete the example folder

We've understood the examples. Now wipe them so the project is clean for our own
models:

```bash
rm -rf models/example
```

Verify:

```bash
ls models/
# (empty)
```

The `models/` folder is now empty. That's the clean slate we want.

## Step 6: Clean up the `dbt_project.yml` references

Open `dbt_project.yml`. Near the bottom you'll see:

```yaml
models:
  my_first_project:
    example:
      +materialized: view
```

The `example:` line refers to the folder we just deleted. It does no harm to leave
it, but for cleanliness, remove the lower two lines so it reads:

```yaml
models:
  my_first_project:
```

Save the file.

(We'll add real configuration to this block later. For now, empty is fine — dbt
will fall back to its built-in default, which is `view`.)

## What did you just do?

- Read two example models and understood what each one produces.
- Met `{{ config(...) }}` and `{{ ref(...) }}` — your first taste of Jinja in dbt.
- Met `schema.yml` and column-level tests.
- Deleted the example folder so we can build our own models from scratch.

## Common confusion

> "Why did dbt put example models if we're just going to delete them?"

For people who want to see dbt do *something* immediately. New users run `dbt
init`, then `dbt run`, see something happen, and that builds confidence. We're
deleting them because they'd just clutter the workspace once you understand them.

> "Should I delete `schema.yml` too?"

We did — `rm -rf models/example` removed the whole folder, schema.yml included.
We'll write our own schema-style yml file later in file 18.

> "What's `{{ config(...) }}`?"

A way to override settings for one specific model. The example model used it to
say "this one should be a table, not a view." You can also set things like
`tags`, `unique_key` (for incremental models), and others.

---

Clean slate. Let's run dbt with nothing in it and see what happens.

← Previous&nbsp; [`08-dbt-debug.md`](08-dbt-debug.md) &nbsp;·&nbsp; Next →&nbsp; [`10-dbt-run.md`](10-dbt-run.md)
