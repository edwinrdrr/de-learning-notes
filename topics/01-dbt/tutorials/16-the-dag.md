# 16 — The DAG: how dbt figures out build order

← Previous&nbsp; [`15-first-mart-model.md`](15-first-mart-model.md) &nbsp;·&nbsp; Next →&nbsp; [`17-why-tests.md`](17-why-tests.md)

---

You just ran `dbt run` and dbt built three models in the right order without
being told. This file explains how dbt knew.

This file is mostly conceptual + a couple of selector examples. ~10 minutes.

## What is a DAG?

DAG stands for **Directed Acyclic Graph**:

- **Graph** — a bunch of nodes connected by lines.
- **Directed** — the lines have arrows (point one way).
- **Acyclic** — no loops (you can't follow arrows back to where you started).

In dbt, the nodes are your models (and seeds/sources/snapshots). The arrows show
*who depends on whom*. Each model is built before anything that depends on it.

Your project's DAG right now looks like this:

```
raw_customers (seed) ──► stg_customers ──┐
                                          ├──► dim_customers
raw_orders (seed) ──────► stg_orders ─────┘
```

`stg_customers` depends on `raw_customers` (because the staging model does `{{
ref('raw_customers') }}`). `dim_customers` depends on both staging models. dbt
builds in topological order — anything with no dependencies first, then anything
whose dependencies are now built, and so on.

## How dbt knows the dependencies

The magic ingredient is `{{ ref(...) }}`. Every time a model contains
`{{ ref('foo') }}`, dbt records "this model depends on foo." After parsing every
file, dbt has the full dependency list. That's the DAG.

Same for `{{ source('group', 'table') }}` — that's also a dependency.

Notice what you DIDN'T do:

- You didn't list dependencies in any config.
- You didn't tell dbt the order.
- You didn't write an `Airflow.dag.py` declaring task order.

dbt **derives the DAG from the SQL itself**. The very act of referencing a model in
your SQL declares the dependency. There's exactly one source of truth — the SQL.
This is why dbt is hard to misuse: if your model runs successfully against a
referenced model, dbt knows the order.

## See the DAG via `dbt list`

```bash
dbt list --resource-type model
```

Output:

```
my_first_project.staging.stg_customers
my_first_project.staging.stg_orders
my_first_project.marts.dim_customers
```

That's the list of models. For the actual DAG visualization, we'll use `dbt docs`
in file 24.

But the build order can be implied by re-running:

```bash
dbt run
```

Look at the output — models 1 and 2 are the staging ones, then model 3 is the
mart. dbt always runs in dependency order.

## Parallelism

Look more carefully at this part of the output:

```
1 of 3 START sql view model dbt_tutorial.stg_customers ... [RUN]
2 of 3 START sql view model dbt_tutorial.stg_orders ........ [RUN]
```

Both staging models START at the same time. That's because:

- `stg_customers` and `stg_orders` don't depend on each other.
- Your profile has `threads: 4`, so dbt is willing to run up to 4 models in
  parallel.

`dim_customers` waits for both to finish — it can't start until both its
dependencies are done.

This is part of why dbt is fast for big projects: it parallelizes everything it
can.

## Selectors that use the DAG

The dbt CLI has powerful `--select` syntax for picking subsets of the DAG. A few
examples:

```bash
# Just one model
dbt run --select dim_customers
```

```bash
# All models in the staging folder
dbt run --select staging
```

```bash
# stg_customers AND everything downstream of it (the + means "and downstream")
dbt run --select stg_customers+
# This runs stg_customers and dim_customers (dim depends on stg).
```

```bash
# dim_customers AND everything upstream (the + before means "and upstream")
dbt run --select +dim_customers
# This runs stg_customers, stg_orders, AND dim_customers.
```

```bash
# Both directions
dbt run --select +dim_customers+
```

```bash
# Exclude a folder
dbt run --exclude marts
```

```bash
# Models that depend on the raw_customers seed
dbt run --select raw_customers+
```

These selectors are extremely useful for big projects (and for CI). They're the
reason `dbt run --select state:modified.body+` (used in Slim CI) is possible —
dbt finds the changed models and adds everything downstream of them, all from the
DAG.

## What happens if you try to create a cycle?

By definition the DAG can't have cycles. dbt enforces this:

If you write:

```sql
-- models/staging/stg_customers.sql
select * from {{ ref('dim_customers') }}
```

(`stg_customers` references `dim_customers` which references back through
`stg_customers` → cycle), dbt fails at parse time with:

```
dbt found a cycle: my_first_project.staging.stg_customers --> my_first_project.marts.dim_customers --> my_first_project.staging.stg_customers
```

That's the protection: dbt won't even let you compile a project with a cycle. You
fix the data flow before dbt will run.

## DAG visualization

If you want to actually *see* the graph as a picture, you have two options:

1. **dbt docs** — `dbt docs generate` then `dbt docs serve` shows an interactive
   graph in your browser. (We'll cover this in file 24.)
2. **Inspect `manifest.json`** — `target/manifest.json` has the full DAG as JSON.
   You can `cat target/manifest.json | jq '.parent_map'` to dump the parent
   relationships (you'll need `jq` installed).

For now, just know it's there. You'll see it in file 24.

## What did you just learn?

- A DAG is a directed acyclic graph: nodes + arrows + no cycles.
- dbt's DAG is built automatically from `{{ ref(...) }}` and `{{ source(...) }}`
  calls in your SQL.
- The build order is the topological order of the DAG.
- dbt runs independent models in parallel up to your `threads` setting.
- `--select` and `--exclude` flags let you pick subsets of the DAG.

## Common confusion

> "Where's the visualization?"

In `dbt docs serve` (file 24). For now, the textual output of `dbt run` shows the
build order, which is enough.

> "What if I have two unrelated pipelines in one project?"

Fine. dbt's DAG will have two disconnected components. Each is independent. You
can `dbt run --select pipeline1` to build just one.

> "Can I make dbt build something in a specific order, regardless of the DAG?"

No, and you shouldn't try. The DAG IS the order. If you want model A before model
B, make B reference A with `ref()`. If that's not natural, you probably don't
actually need that ordering — you're trying to express something else.

> "What about non-SQL dependencies? E.g., I want a model to run after a non-dbt
> Python script finishes."

That's not dbt's job. dbt handles model-to-model order. For cross-tool ordering,
use an orchestrator like Airflow that runs dbt as one step among many.

> "Can I see the DAG before running anything?"

Yes — `dbt parse` then `dbt list --select +my_model` shows everything `my_model`
depends on. You can also open `target/manifest.json` if you really want to dig.

---

✅ **Part B checkpoint** — you've built a working pipeline: seeds → staging → mart.
You understand how dbt computes the build order. Time to add tests so you can
trust the result.

← Previous&nbsp; [`15-first-mart-model.md`](15-first-mart-model.md) &nbsp;·&nbsp; Next →&nbsp; [`17-why-tests.md`](17-why-tests.md)
