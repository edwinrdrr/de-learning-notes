# 22 — Materializations

← Previous&nbsp; [`21-dbt-build.md`](21-dbt-build.md) &nbsp;·&nbsp; Next →&nbsp; [`22b-incremental-built.md`](22b-incremental-built.md)

---

You've already used two materializations without thinking about it much: your
staging models built as **views**, your `dim_customers` mart built as a **table**.
This file explains the four materializations dbt offers and when to use each.

## What does "materialize" mean?

A model is just SQL — a SELECT statement that produces rows. "Materializing" it
is dbt's term for **how that SQL gets stored in the warehouse**:

- **view** — store the SELECT as a virtual table. Rows are computed on every query.
- **table** — execute the SELECT once, store the result as a physical table. Rows
  are pre-computed.
- **incremental** — first run, build as a table. Subsequent runs, only add new
  rows.
- **ephemeral** — don't store at all. The model becomes a CTE inlined into models
  that depend on it.

The choice trades off between **build cost**, **query cost**, **freshness**, and
**storage**.

## view

```sql
{{ config(materialized='view') }}
```

What dbt sends to BigQuery:
```sql
create or replace view ... as
select ... from ...
```

A view is metadata — no data. The SELECT runs each time you (or another model)
query the view.

**Use views when**:

- The underlying transformation is cheap (a few simple operations on a small table).
- You want guaranteed-fresh results (no rebuild needed when underlying data changes).
- Storage matters more than query cost.
- It's a staging-layer model (light transformations on raw data; cheap to recompute).

**Avoid views when**:

- The transformation is expensive (heavy joins, aggregations) and queried often —
  recomputing each query is wasteful.

This is why our staging models are views: cheap renames over a small seed.

## table

```sql
{{ config(materialized='table') }}
```

What dbt sends:
```sql
create or replace table ... as
select ... from ...
```

dbt runs the SELECT once, stores the rows physically. Subsequent queries on the
table just read those rows back — no recomputation.

**Use tables when**:

- The transformation is expensive (joins, aggregations) AND the result is queried
  many times (dashboards, BI tools).
- You want a consistent snapshot — the underlying data may change, but the table
  is fixed until the next dbt build.
- It's a mart-layer model (analytics-ready, queried often).

**Avoid tables when**:

- The transformation is so cheap that views work fine.
- Build time matters more than query time.
- You need real-time freshness (tables are only as fresh as the last `dbt build`).

This is why `dim_customers` is a table: a JOIN + COALESCE that BI dashboards will
hit many times. Compute once on build, then queries are fast.

## incremental

```sql
{{ config(
    materialized='incremental',
    unique_key='id'
) }}

select * from {{ ref('source_data') }}
{% if is_incremental() %}
  where event_ts > (select max(event_ts) from {{ this }})
{% endif %}
```

What dbt does:

- **First run**: builds the model as a table from scratch (full SELECT).
- **Subsequent runs**: only inserts/merges rows that didn't exist before.

You provide a `unique_key` (the column that identifies each row) and a `where`
clause guarded by `{% if is_incremental() %}` — that clause filters to "new rows
only" on subsequent runs.

The `{{ this }}` Jinja variable refers to the current table — so the WHERE checks
"rows where the timestamp is newer than anything already in this table."

**Use incremental when**:

- The model is over a huge fact table (millions to billions of rows).
- Most rows don't change after they're first written (append-only-ish event data).
- Full rebuilds would take too long.

**Avoid incremental when**:

- The dataset is small enough that a full table rebuild is fine. The complexity
  isn't worth it.
- Rows change after first write (mutable data). You'd need to think hard about
  the merge strategy — often easier to just rebuild.

We're NOT using incremental in this tutorial. With 10 rows, it'd be overkill.

The two things you'll trip on later when you actually use incremental:

1. **`on_schema_change`**: when you add a new column, what does dbt do? Default is
   `ignore` — silently drops the new column on incremental runs. You probably want
   `append_new_columns`:
   ```sql
   {{ config(
       materialized='incremental',
       unique_key='id',
       on_schema_change='append_new_columns'
   ) }}
   ```

2. **`dbt run --full-refresh`**: forces a complete rebuild even on an incremental
   model. Useful when you've changed the model's logic and want to recompute from
   scratch.

## ephemeral

```sql
{{ config(materialized='ephemeral') }}
```

What dbt does: **nothing in the warehouse**. The model's SQL gets inlined as a CTE
into any model that `ref()`s it.

So if `model_B` does `select * from {{ ref('ephemeral_A') }}`, the compiled SQL of
`model_B` includes ephemeral_A's full SELECT as a CTE.

**Use ephemeral when**:

- You have shared sub-query logic that 2-3 other models reuse.
- You don't want to materialize the intermediate result in the warehouse.
- It's small and cheap (since it gets inlined every time).

**Avoid ephemeral when**:

- The shared logic is expensive (it'll be re-executed in every downstream model).
- It's used by many models (the inlining bloats the compiled SQL).
- You need to query it directly (you can't — there's nothing in the warehouse).

Ephemeral is a niche tool. Most projects don't use it. Stick to view/table/incremental
99% of the time.

## Quick decision flow

```
Is this a staging model?      → view
Is this a mart, queried often? → table
Is the table HUGE and you can't afford a full rebuild? → incremental
Is this a reusable subquery you don't want in the warehouse? → ephemeral
Default: view for staging, table for marts.
```

## A worked example

Imagine: `fct_orders` is your largest fact table, 10 million rows. Every dbt build,
you rebuild it from scratch. Each build takes 8 minutes.

You switch to incremental:

```sql
{{ config(
    materialized='incremental',
    unique_key='order_id',
    on_schema_change='append_new_columns'
) }}

select * from {{ ref('stg_orders') }}
{% if is_incremental() %}
  where order_date > (select max(order_date) from {{ this }})
{% endif %}
```

After this change, full rebuilds (first run, or `--full-refresh`) still take 8
minutes. But subsequent normal builds — which only need to add yesterday's new
orders — take 20 seconds. That's the win.

The tradeoff: more complexity in the model file, plus the configuration knowledge
(unique_key, the is_incremental conditional, on_schema_change).

## What did you just learn?

- The four materializations: view, table, incremental, ephemeral.
- What each one does on the BigQuery side.
- When to use each.
- Why staging is usually view and marts are usually table.
- A peek at incremental for big fact tables.

## Common confusion

> "Can I switch materialization later?"

Yes. Change `{{ config(materialized='X') }}` and re-run. dbt drops the old
form and creates the new one. Small disruption (existing queries fail until the new
table/view exists), but possible.

> "Why is `incremental` mentioned so much in dbt talks if I shouldn't use it
> often?"

Because the people writing those talks work at companies with 100GB+ tables
where full rebuilds are infeasible. For our tutorial with 10 rows, full rebuilds
are instant. Incremental is a tool that earns its complexity when you have the
problem it solves — and not before.

> "What's the difference between ephemeral and a CTE inside one model?"

A CTE is local to one model. Ephemeral is shared logic that multiple models can
inline. If only one model needs it, just use a CTE. If three models need the same
subquery, ephemeral lets you DRY it up.

> "Does materialization affect tests?"

No. Tests run after the model exists in BigQuery (or is inlined, for ephemeral).
The materialization choice doesn't change which tests apply or whether they pass.

## BigQuery-specific configs: partitioning and clustering

Materialization choice (view/table/incremental/ephemeral) is the warehouse-agnostic
decision. But on BigQuery there are two more knobs that determine how a TABLE or
INCREMENTAL model **physically stores** rows — and they matter enormously for
cost and performance:

### `partition_by`

```sql
{{ config(
    materialized='table',
    partition_by={
        'field': 'order_date',
        'data_type': 'date',
        'granularity': 'day'
    }
) }}
```

What BigQuery does: physically splits the table into one partition per day. When
a query has `where order_date = '2026-01-01'`, BQ only scans that one partition.

**Cost impact:** BigQuery's pricing is per byte scanned. A 100 GB table queried
with `where order_date = X` might scan 100 MB instead of 100 GB if partitioned.
That's 1000× cheaper.

Common partition columns: `event_date`, `order_date`, `created_at` (use
`granularity: 'day'` for date, `'hour'` for timestamp, `'month'` for older
columns where you don't query a single day).

### `cluster_by`

```sql
{{ config(
    materialized='table',
    partition_by={'field': 'order_date', 'data_type': 'date'},
    cluster_by=['customer_id', 'status']
) }}
```

Within each partition, BQ sorts and groups rows by the cluster columns. When you
filter on a clustered column, BQ can skip blocks within the partition.

**Use clustering on:** columns with high cardinality that you filter on often.
`customer_id` is the canonical example.

### Combined: the standard mart pattern

```sql
{{ config(
    materialized='incremental',
    unique_key='order_id',
    on_schema_change='append_new_columns',
    partition_by={
        'field': 'order_date',
        'data_type': 'date',
        'granularity': 'day'
    },
    cluster_by=['customer_id']
) }}
```

Big fact table, partitioned by date for cheap date-range queries, clustered by
customer_id for cheap per-customer queries, incremental so daily builds only
process yesterday's data. This is the standard production pattern for
high-volume BQ fact tables.

For our 10-row tutorial it's overkill. For a real project with millions of rows,
forgetting partition_by means the BQ bill is 100× too high.

---

You understand the four materializations and BigQuery's physical-layout knobs.
Now let's actually build an incremental for real — that's where most of the
gotchas live.

← Previous&nbsp; [`21-dbt-build.md`](21-dbt-build.md) &nbsp;·&nbsp; Next →&nbsp; [`22b-incremental-built.md`](22b-incremental-built.md)
