# 13 — Your first staging model

← Previous&nbsp; [`12-jinja-and-ref.md`](12-jinja-and-ref.md) &nbsp;·&nbsp; Next →&nbsp; [`14-layered-architecture.md`](14-layered-architecture.md)

---

Time to write a real model. We'll do the simplest useful kind: a **staging model**
that takes a raw table and applies light cleanup.

## What is a "staging" model?

By convention, the first layer of dbt models is called **staging**. The purpose of
staging is:

- Rename columns (drop the `raw_` prefix, e.g., `raw_customers.id` → `customer_id`).
- Cast types if needed.
- Drop columns you don't care about downstream.
- Standardize naming (snake_case everywhere, consistent casing).

What staging models DON'T do:

- Join tables (that's marts).
- Aggregate (that's marts).
- Apply business logic (that's marts).
- Filter to a subset (mostly that's marts; sometimes staging filters obvious junk).

The rule: **one staging model per source table.** If you have 5 raw tables, you'll
have 5 staging models. Each one is a 1:1 cleanup pass.

## Step 1: Make the staging folder

We deleted `models/example/` earlier. The folder is empty. We'll organize models
into subfolders. The convention is `models/staging/` for staging models.

```bash
mkdir -p models/staging
```

## Step 2: Write `stg_customers.sql`

Create a new file:

```bash
cat > models/staging/stg_customers.sql <<'EOF'
with src as (
    select * from {{ ref('raw_customers') }}
)

select
    id           as customer_id,
    first_name,
    last_name,
    signup_date
from src
EOF
```

Read the file:

```bash
cat models/staging/stg_customers.sql
```

Let's walk through it.

### `with src as (...)`

This is a **CTE** (Common Table Expression). It names a subquery `src`. Equivalent
plain English: "Define `src` as the result of selecting everything from
raw_customers." Then we can refer to `src` in the next part.

dbt convention is to wrap the first query (the raw source) in a CTE, then operate
on it. This is a stylistic preference but a popular one — makes the model's
structure easy to scan.

### `from {{ ref('raw_customers') }}`

This is the `ref()` we just learned. It tells dbt: "find the thing named
`raw_customers` (which is the seed) and substitute its fully-qualified table
name."

When dbt compiles this, it becomes:
```sql
from `sweet-mountain-471812`.`dbt_tutorial`.`raw_customers`
```

### The `select` block

```sql
select
    id           as customer_id,
    first_name,
    last_name,
    signup_date
from src
```

This is the actual transformation. Two things happening:

1. **Rename `id` to `customer_id`.** This is a staging convention: name primary
   keys descriptively so when you join tables in marts, you don't have ambiguous
   `id` columns colliding.
2. **Pass through everything else** unchanged (`first_name`, `last_name`,
   `signup_date`).

We could do more — cast types, drop columns — but for this raw data, a rename is
enough.

## Step 3: Write `stg_orders.sql`

Same pattern. Create:

```bash
cat > models/staging/stg_orders.sql <<'EOF'
with src as (
    select * from {{ ref('raw_orders') }}
)

select
    id           as order_id,
    customer_id,
    order_date,
    status,
    amount
from src
EOF
```

Same logic: rename `id` → `order_id`, pass everything else through.

Verify both files exist:

```bash
ls models/staging/
# stg_customers.sql  stg_orders.sql
```

## Step 4: Compile to see the SQL

Before running, let's compile. This step is optional for normal use, but it's a
good debug habit:

```bash
dbt compile --select staging
```

Look at the rendered SQL:

```bash
cat target/compiled/my_first_project/models/staging/stg_customers.sql
```

You'll see:

```sql
with src as (
    select * from `sweet-mountain-471812`.`dbt_tutorial`.`raw_customers`
)

select
    id           as customer_id,
    first_name,
    last_name,
    signup_date
from src
```

The `{{ ref('raw_customers') }}` got replaced with the actual full table name.
That's the SQL BigQuery will see.

## Step 5: Run the staging models

```bash
dbt run --select staging
```

Expected output:

```
Running with dbt=1.11.x
Found 2 models, ...

1 of 2 START sql view model dbt_tutorial.stg_customers ........ [RUN]
2 of 2 START sql view model dbt_tutorial.stg_orders ............ [RUN]
1 of 2 OK created sql view model dbt_tutorial.stg_customers ... [CREATE VIEW (0 rows...)]
2 of 2 OK created sql view model dbt_tutorial.stg_orders ....... [CREATE VIEW (0 rows...)]

Finished running 2 view models in X.XXs.
```

Two important things in this output:

- **`view`**: dbt materialized these as views, not tables. This is the project
  default (no `+materialized:` set anywhere). Views are cheap — they don't store
  rows; they're a saved SELECT statement that runs when queried.
- **`CREATE VIEW`**: the actual SQL command dbt sent to BigQuery.

## Step 6: Verify in BigQuery

In the BigQuery console, expand your `dbt_tutorial` dataset. You'll now see:

- `raw_customers` (table) — the seed
- `raw_orders` (table) — the seed
- `stg_customers` (view) — your staging model
- `stg_orders` (view) — your staging model

Click `stg_customers` and preview — you'll see 10 rows. The `id` column is now
called `customer_id`. The other columns are unchanged.

You can also query:

```sql
SELECT customer_id, first_name FROM `your-project-id.dbt_tutorial.stg_customers` LIMIT 3
```

## Why staging models are views (not tables)

A view in BigQuery is just a saved SELECT. When you query a view, BigQuery runs
the underlying SELECT on the fly.

That's perfect for staging:
- Staging is cheap transformation. Recomputing on each query is fine.
- It's always fresh — if the underlying seed (or in real projects, the ingested
  raw table) updates, staging reflects it immediately, no rebuild needed.
- It saves storage (a view is metadata, not data).

For marts (the next layer), we'll typically use tables instead. Marts are queried
heavily; you don't want to recompute them every time.

## What did you just do?

- Wrote two staging models — small, single-purpose, one per raw table.
- Used `{{ ref('seed_name') }}` to point at the seeds.
- Compiled and ran. dbt created views in BigQuery.
- Verified the views are queryable.

## Common confusion

> "Why wrap the model in `with src as (select * from ...)`?"

You don't have to. You could write:

```sql
select id as customer_id, first_name, last_name, signup_date
from {{ ref('raw_customers') }}
```

It works identically. The CTE is a stylistic choice that makes longer models
easier to read — when you have multiple CTEs (joining, aggregating), the structure
is consistent. For this tiny model, it's overkill. For larger ones, you'll be glad
you have it.

> "What does 'view' mean in BigQuery specifically?"

A view is a saved SELECT statement that BigQuery treats as a virtual table. When
you query the view, BigQuery substitutes the underlying SELECT and runs it. No
data is stored.

> "I see `(0 rows)` in the dbt output for my view. Did it not work?"

`0 rows` is correct for `CREATE VIEW` — views don't store rows. The view is the
SELECT statement; the rows are computed at query time. If you query the view
(`SELECT COUNT(*) FROM stg_customers`), you'll get 10.

> "Should I name my models `stg_customers` or `staging_customers`?"

Convention is `stg_` (3-letter prefix). Same for mart fact tables (`fct_`) and
dimension tables (`dim_`). These prefixes make the model layer obvious at a
glance.

---

We have two staging models. Before we join them into a mart, let's step back and
understand the layered-architecture pattern this is part of.

← Previous&nbsp; [`12-jinja-and-ref.md`](12-jinja-and-ref.md) &nbsp;·&nbsp; Next →&nbsp; [`14-layered-architecture.md`](14-layered-architecture.md)
