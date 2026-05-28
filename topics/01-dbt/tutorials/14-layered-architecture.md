# 14 — The layered architecture (sources → staging → marts)

← Previous&nbsp; [`13b-sources.md`](13b-sources.md) &nbsp;·&nbsp; Next →&nbsp; [`15-first-mart-model.md`](15-first-mart-model.md)

---

You have two staging models. In file 15 you'll build a mart that joins them. Before
that — a slight pause to explain the convention that's about to make a lot of sense.

This file is conceptual. No commands to type. ~5 minutes.

## The shape

Most well-organized dbt projects have three layers:

```
        sources                    (raw tables, loaded by ingestion or seeds)
            │
            ▼
       staging                     (light cleanup, 1:1 with sources)
            │
            ▼
        marts                      (analytics-ready: joins, aggregations, business logic)
```

Data flows downward. Each layer transforms the one above.

## Why three layers, not one?

You *could* skip staging and go directly from sources to marts. People do this
when starting out. It seems efficient — fewer files, less indirection.

But here's what happens at month 3:

1. Your ingestion tool renames a column (`raw_orders.id` → `raw_orders.order_id`).
2. Your raw column was used directly in 12 mart models that each have their own
   joins and business logic.
3. You now edit 12 files to fix the rename, hoping you don't break any of them.

With staging:

1. Same ingestion change happens.
2. You edit **one** file: `stg_orders.sql` (the staging model).
3. The 12 marts that reference `stg_orders.order_id` continue to work unchanged.

The cost of the extra layer is small (one file per raw table, all almost identical).
The payoff is huge as soon as anything upstream changes — and **something upstream
always changes**.

## What each layer is for

### Sources

**Sources are the raw data**. They live in the warehouse but **dbt doesn't build
them**. They got there via:

- An ingestion tool (Fivetran, Airbyte, Stitch).
- A custom script (`snapshot.py` in spotify-pipeline).
- A manual upload.
- Seeds (CSVs in `seeds/` — our case for the tutorial).

In a `_sources.yml` file, you tell dbt "these tables exist; here's where; here's
what columns they have." You can also declare tests on them (`not_null` on
`order_id` at the source layer catches ingestion bugs).

You reference sources with `{{ source('source_group', 'table_name') }}`.

> In **this tutorial** we're using seeds as our "source" stand-ins, and referencing
> them with `{{ ref('raw_customers') }}` instead of `source(...)`. The pattern is
> the same; only the function name differs.

### Staging

**Staging is light cleanup, 1:1 with sources**. The rules:

- **One staging model per source table.** No exceptions.
- **No joins.** If you find yourself joining in staging, you're doing mart work
  too early.
- **No aggregations.** Same reason.
- **Materialized as `view`** (cheap; staging is just a saved SELECT).
- **Renames** are common: `id` → `customer_id` so joins downstream are unambiguous.
- **Casts** are common: a string-typed amount becomes a numeric.
- **Dropping garbage columns** is common: most raw tables have a few you'll never
  use.

You reference staging models with `{{ ref('stg_customers') }}`.

You already did all of this in file 13.

### Marts

**Marts are the analytics-ready layer.** This is what dashboards, analysts, and
downstream applications consume. The rules:

- **Built from staging, never from sources.** Always `ref()` a staging model, not a
  source. If you find yourself needing to source a table that isn't in staging
  yet, add a staging model for it first.
- **Joins, aggregations, business logic** happen here.
- **Materialized as `table`** (expensive computation, queried often — cache the
  result).
- Often split into two flavors:
  - **Facts (`fct_*`)** — event-level. One row per order, one row per click, one
    row per shipment.
  - **Dimensions (`dim_*`)** — entity-level. One row per customer, one row per
    product, one row per store.

You reference marts as needed (`{{ ref('dim_customers') }}` in another mart if
you're composing).

## What you'll build next

In file 15 we'll build `dim_customers` — a dimensional mart with one row per
customer that includes:
- Customer demographics (`first_name`, `last_name`, `signup_date`) from
  `stg_customers`.
- Order statistics (`first_order_date`, `total_spent`, `orders_count`) computed
  from `stg_orders`.

That's a typical dimensional mart shape: one entity, enriched with
pre-aggregated metrics about it.

## The whole picture for this tutorial

When we're done, the project will have:

```
seeds/                                    ← "raw" CSVs (sub for ingestion)
├── raw_customers.csv
└── raw_orders.csv
     │
     ▼ (dbt seed)
[BigQuery: raw_customers, raw_orders tables]
     │
     ▼ (dbt run, models/staging/)
models/staging/
├── stg_customers.sql    →    [BQ view: stg_customers]
└── stg_orders.sql       →    [BQ view: stg_orders]
                                    │
                                    ▼ (dbt run, models/marts/)
                              models/marts/
                              └── dim_customers.sql  →  [BQ table: dim_customers]
```

That's a full pipeline. Tiny by real-world standards (one mart!) but it has every
ingredient.

## Folder naming convention

Inside `models/`, the convention is:

```
models/
├── staging/
│   ├── stg_customers.sql
│   ├── stg_orders.sql
│   └── _sources.yml         ← (or _stg_models.yml, etc.)
└── marts/
    ├── dim_customers.sql
    └── _marts_models.yml
```

Some projects add a `models/intermediate/` layer between staging and marts for
*reusable transformations* (e.g., a customer-with-order-totals subquery that's
used by 3 different marts). That's an optimization for large projects; skip it
until you need it.

## Naming convention summary

| Prefix | Layer | Materialization | Purpose |
|---|---|---|---|
| `stg_` | staging | view | Light cleanup, 1:1 with source |
| `int_` | intermediate (optional) | view or ephemeral | Reusable sub-transformations |
| `dim_` | marts (dimensions) | table | One row per entity |
| `fct_` | marts (facts) | table or incremental | One row per event |

These are conventions, not laws. You'll see variations. But picking a consistent
naming scheme makes the project navigable.

## What did you just learn?

- The three-layer convention: sources → staging → marts.
- Why each layer exists and what rules each follows.
- Which materialization is typical for each (view / view / table).
- Naming prefixes (`stg_`, `dim_`, `fct_`) and what they mean.
- Where the full data flow ends up: from seeds to a final dim table.

## Common confusion

> "Do I have to use this structure?"

No. dbt doesn't enforce it. But every real-world project you'll read uses something
close to this. Adopting it early means you won't refactor later. spotify-pipeline,
the canonical dbt examples, dbt Labs's own projects — all use this shape.

> "What about the intermediate layer?"

You don't need it for most projects. Add it when you find yourself writing the
same complex CTE in three different mart models — that's the signal to factor it
into an intermediate model. Until then, the two-layer (staging → marts) shape is
enough.

> "Can a mart reference another mart?"

Yes, but be careful. A mart that depends on another mart creates a longer
dependency chain. If `mart_a` references `mart_b`, then `mart_a` rebuilds whenever
`mart_b` does. For dashboards and BI tools, this can be fine. For complex
pipelines, prefer to keep marts shallow — each mart should be cheap to rebuild on
its own.

> "What's the difference between a fact and a dimension?"

Facts: event-level (orders, sales, clicks). Lots of rows. Numeric metrics.
Dimensions: entity-level (customers, products, dates). Fewer rows. Categorical
attributes.

A typical join: `fct_orders` (one row per order) joins to `dim_customers` (one
row per customer) on `customer_id`. The fact answers "what happened"; the
dimension answers "to whom / about what."

---

Mental model in hand. Let's build the mart.

← Previous&nbsp; [`13b-sources.md`](13b-sources.md) &nbsp;·&nbsp; Next →&nbsp; [`15-first-mart-model.md`](15-first-mart-model.md)
