# 15 — Your first mart model

← Previous&nbsp; [`14-layered-architecture.md`](14-layered-architecture.md) &nbsp;·&nbsp; Next →&nbsp; [`16-the-dag.md`](16-the-dag.md)

---

Time to write a mart that joins our two staging models. We'll build `dim_customers`:
**one row per customer, enriched with order statistics**.

## What the mart should produce

For each customer in `stg_customers`, we want a row with:

- The customer's identity (`customer_id`, `first_name`, `last_name`, `signup_date`).
- Their order statistics: when they first ordered, when they most recently ordered,
  how many completed orders they have, and how much they spent in total. Only
  count *completed* orders (`status = 'completed'`).
- Customers with zero completed orders should still appear, with `orders_count =
  0` and `total_spent = 0`.

That last point is important: we're describing customers, not orders. A customer
without orders is still a customer.

## Step 1: Make the marts folder

```bash
mkdir -p models/marts
```

## Step 2: Write `dim_customers.sql`

```bash
cat > models/marts/dim_customers.sql <<'EOF'
{{ config(materialized='table') }}

with customers as (
    select * from {{ ref('stg_customers') }}
),

orders as (
    select * from {{ ref('stg_orders') }}
),

customer_orders as (
    select
        customer_id,
        min(order_date)  as first_order_date,
        max(order_date)  as most_recent_order_date,
        count(*)         as orders_count,
        sum(amount)      as total_spent
    from orders
    where status = 'completed'
    group by customer_id
)

select
    c.customer_id,
    c.first_name,
    c.last_name,
    c.signup_date,
    co.first_order_date,
    co.most_recent_order_date,
    coalesce(co.orders_count, 0)  as orders_count,
    coalesce(co.total_spent,  0)  as total_spent
from customers c
left join customer_orders co using (customer_id)
EOF
```

Let's walk through it from the top.

### `{{ config(materialized='table') }}`

We saw this in the example. It overrides the project default of `view` for this
specific model. Marts are typically tables — they're queried frequently, so we want
the result physically stored rather than recomputed every time someone queries.

### CTE block 1: `customers`

```sql
customers as (
    select * from {{ ref('stg_customers') }}
)
```

Brings in the staging customers. We could have referenced `stg_customers` directly
in the final SELECT, but pulling it into a CTE makes the model self-documenting
("here's the customers input").

### CTE block 2: `orders`

Same pattern, for the orders staging model.

### CTE block 3: `customer_orders` — the aggregation

This is the meat:

```sql
customer_orders as (
    select
        customer_id,
        min(order_date)  as first_order_date,
        max(order_date)  as most_recent_order_date,
        count(*)         as orders_count,
        sum(amount)      as total_spent
    from orders
    where status = 'completed'
    group by customer_id
)
```

Reading this in English:

- For each `customer_id`:
  - `min(order_date)` — the earliest order date for that customer
  - `max(order_date)` — the latest
  - `count(*)` — how many orders
  - `sum(amount)` — total amount
- Only consider rows where `status = 'completed'`.

After this CTE runs, `customer_orders` has one row per customer who has ANY
completed orders. Customers with zero completed orders are absent.

### Final SELECT — the LEFT JOIN

```sql
select
    c.customer_id,
    c.first_name,
    c.last_name,
    c.signup_date,
    co.first_order_date,
    co.most_recent_order_date,
    coalesce(co.orders_count, 0)  as orders_count,
    coalesce(co.total_spent,  0)  as total_spent
from customers c
left join customer_orders co using (customer_id)
```

Two things to notice:

**1. `LEFT JOIN`.** We start from `customers` and *left-join* in `customer_orders`.
If a customer has no completed orders, the join produces a row where the
`customer_orders` columns are NULL.

**2. `COALESCE(co.orders_count, 0)`.** If `customer_orders` had no match (the
customer has no orders), the joined row will have NULL for those columns.
`COALESCE` says "if NULL, substitute 0 instead." That's how we get "0 orders" for
customers who haven't ordered, instead of NULL.

### `using (customer_id)`

`USING (col)` is shorthand for `ON c.col = co.col` when both tables share a column
name. Cleaner than the verbose form.

## Step 3: Compile and inspect

```bash
dbt compile --select dim_customers
```

Look at what dbt rendered:

```bash
cat target/compiled/my_first_project/models/marts/dim_customers.sql
```

You'll see the same SQL but with the `{{ ref(...) }}` calls replaced by full table
paths. That's the SQL BigQuery will execute.

## Step 4: Run it

```bash
dbt run --select dim_customers
```

Expected:

```
1 of 1 START sql table model dbt_tutorial.dim_customers ... [RUN]
1 of 1 OK created sql table model dbt_tutorial.dim_customers . [CREATE TABLE (10.0 rows, ...)]

Finished running 1 table model in X.XXs.
```

Note **`CREATE TABLE`** (not `CREATE VIEW`) and **`10.0 rows`** — physically
stored.

## Step 5: Query the result

In the BigQuery console:

```sql
SELECT
    customer_id,
    first_name,
    last_name,
    orders_count,
    total_spent
FROM `your-project-id.dbt_tutorial.dim_customers`
ORDER BY total_spent DESC
LIMIT 5
```

You should see something like:

```
customer_id | first_name | last_name | orders_count | total_spent
─────────────────────────────────────────────────────────────────
1           | Michael    | Phillips  | 3            | 39
5           | Katherine  | Rivera    | 1            | 17
7           | Martin     | Long      | 1            | 5
3           | Kathleen   | Payne     | 1            | 20
8           | Frank      | Owens     | 0            | 0
...
```

Note customer 8 (Frank Owens) — `orders_count = 0`, `total_spent = 0`. That's
because Frank has no rows in `raw_orders` at all. The LEFT JOIN + COALESCE
preserved him in the dimensional table.

If you ran a normal `INNER JOIN`, Frank would have been dropped.

## Step 6: Run everything together

To this point we've run staging and marts separately. Now run the whole project:

```bash
dbt run
```

Expected:

```
Running with dbt=1.11.x
Found 3 models, ...

1 of 3 START sql view model dbt_tutorial.stg_customers ........ [RUN]
2 of 3 START sql view model dbt_tutorial.stg_orders ............ [RUN]
1 of 3 OK created sql view model dbt_tutorial.stg_customers ... [CREATE VIEW]
2 of 3 OK created sql view model dbt_tutorial.stg_orders ....... [CREATE VIEW]
3 of 3 START sql table model dbt_tutorial.dim_customers ........ [RUN]
3 of 3 OK created sql table model dbt_tutorial.dim_customers .. [CREATE TABLE (10.0 rows, ...)]

Finished running 2 view models, 1 table model in X.XXs.
```

dbt:
1. Recognized 3 models total.
2. Ran the 2 staging models first (in parallel — they don't depend on each other).
3. Then ran `dim_customers` (it depends on both staging models).

You never told dbt the order. It figured it out from the `{{ ref(...) }}` calls.

That's the DAG in action — which is exactly what file 16 is about.

## What did you just do?

- Wrote your first mart model: a customer dimension with order statistics.
- Used CTEs to structure the model into readable blocks.
- Applied an aggregation (`GROUP BY customer_id` with multiple aggregate functions).
- Used `LEFT JOIN` + `COALESCE` to preserve customers without orders.
- Overrode the materialization to `table` with `{{ config(materialized='table') }}`.
- Ran the full project and saw dbt build models in dependency order.

## Common confusion

> "Why didn't dbt complain that `customer_orders` doesn't exist as a `{{ ref(...)
> }}`?"

`customer_orders` is a CTE inside the same model file. It's defined locally,
scoped to that SQL statement. CTEs don't need refs. Only when you reference
something built by *another* dbt model or seed do you need `ref()`.

> "I get an error: `Column ... not found`. What gives?"

Probably a typo in a column name. Look at the rendered SQL in
`target/compiled/.../dim_customers.sql` and trace which column is missing in which
table. The actual error message from BigQuery usually points at a line number in
the rendered file.

> "Why CTEs instead of subqueries?"

Style. Both work identically — BigQuery's planner sees through the CTEs. CTEs are
generally more readable for non-trivial queries because each block has a name,
and you can read the structure top-to-bottom.

> "Should this model be `dim_customers` or `customers`?"

dbt naming conventions use the `dim_` prefix for dimensional models. It's a flag
to readers: "this represents an entity, one row per X." `customers` would also
work but loses that context.

> "What if I want to test this model?"

Coming up in Part C (files 17-21). You'll add tests that verify uniqueness on
`customer_id`, non-null on key columns, and a relationships test back to staging.

---

We have a full pipeline running. Now let's actually look at the DAG dbt built.

← Previous&nbsp; [`14-layered-architecture.md`](14-layered-architecture.md) &nbsp;·&nbsp; Next →&nbsp; [`16-the-dag.md`](16-the-dag.md)
