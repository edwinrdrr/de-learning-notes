# 24 — Documentation and the lineage graph

← Previous&nbsp; [`23d-compiled-sql.md`](23d-compiled-sql.md) &nbsp;·&nbsp; Next →&nbsp; [`25-common-errors.md`](25-common-errors.md)

---

dbt can generate a documentation website from your project. It includes every
model's description, every column's description, every test, and an interactive
**lineage graph** showing the DAG. Let's generate it and look around.

## Step 1: Generate the docs

```bash
dbt docs generate
```

What dbt does:

1. Parses your project (same as `dbt parse`).
2. Queries BigQuery for table metadata (column types, row counts, sizes).
3. Builds a `catalog.json` file in `target/` with everything dbt knows.

Expected output (paraphrased):

```
Building catalog
Catalog written to /home/edwin/.../target/catalog.json
```

You now have everything dbt needs to render the docs. But `catalog.json` is just
data — you need a UI to browse it.

## Step 2: Serve the docs

```bash
dbt docs serve
```

dbt starts a local web server (default port 8080) and tries to open your browser
automatically. If the browser doesn't open: navigate to **http://localhost:8080**.

The terminal will show:

```
Serving docs at 8080
To access from your browser, navigate to: http://localhost:8080
Press Ctrl+C to exit.
```

The server stays running until you Ctrl+C.

## Step 3: Click around

You'll see the dbt docs UI. Three main areas:

### Left panel: project tree

Expand `my_first_project` → `models`. You'll see:

```
my_first_project
├── marts
│   └── dim_customers
└── staging
    ├── stg_customers
    └── stg_orders
```

Click `dim_customers`.

### Main panel: model details

For each model:

- **Description** (from your `_marts_models.yml`).
- **Columns** — name, type, description, tests applied.
- **Code** tab — the rendered SQL.
- **References** — what this model depends on (and what depends on it).

This is what your `description:` fields and column docs were for. They show up
here. The richer your YAML, the better the docs.

### Bottom-right: the lineage graph

Look for a small blue circle icon in the bottom-right corner. Click it.

The lineage graph opens. You'll see the DAG visualized:

```
raw_customers ─┬─ stg_customers ─┬─ dim_customers
               │                  │
raw_orders ────┴─ stg_orders ─────┘
```

Each node is a model, seed, or source. Arrows show dependencies. Click any node
to see its details.

You can:
- Zoom (mouse wheel).
- Pan (click and drag).
- Filter to upstream / downstream of a node.
- Select multiple nodes and see what's between them.

This graph is exactly what `dbt` knows. It's the DAG — visualized.

## Step 4: Read what's documented

Navigate to `stg_orders`. You'll see:

- Description: "One row per order, after light staging cleanup." (from your YAML)
- Columns:
  - `order_id` — "Primary key — the order's unique id." Tests: not_null, unique.
  - `customer_id` — "Foreign key into stg_customers." Tests: not_null, relationships.
  - etc.

**This is exactly the YAML you wrote in file 18.** Every `description:` field
you typed shows up here. The `data_tests:` blocks become the tests listed under
each column. Your YAML wasn't just configuration — it was the source of truth
for documentation, all along.

The lesson: the more effort you put into writing good descriptions in YAML, the
better the docs. A model with a one-line description and zero column descriptions
shows up as a near-empty page. A well-documented model becomes a navigable
reference.

## Step 5: Stop the server

Back in your terminal, press `Ctrl+C`. The server stops; the browser tab still
works for a moment but won't reload anything.

## When to use docs

Docs are most useful for:

1. **Onboarding new team members** — they explore the docs site to understand the
   project before reading SQL.
2. **Analysts/stakeholders** — non-engineers can browse without git access.
3. **Yourself in 6 months** — you'll forget what `dim_customers.first_order_date`
   means; the description will remind you.
4. **Sharing with stakeholders** — many teams deploy the docs site to internal
   URLs (e.g., `https://dbt-docs.mycompany.com`).

## Hosting the docs

For team use, generate the docs in CI and deploy them somewhere everyone can
access:

```bash
dbt docs generate
# upload target/manifest.json + target/catalog.json + the static UI files
# (the UI files come from dbt's package; you can find them or use dbt Cloud)
```

There are also third-party tools (dbt Cloud, datafold, etc.) that host the docs
site for you. For solo/learning, `dbt docs serve` locally is fine.

## What you've documented

Looking back at the YAML you wrote:

- Every model has a description.
- Every key column has a description.
- Every test is named and listed.

This is the minimum useful documentation. spotify-pipeline's `dbt/models/marts/_spotify__marts_models.yml`
has a longer description for the mart model — go look if curious.

## What did you just do?

- Ran `dbt docs generate` to build the docs data.
- Ran `dbt docs serve` to spin up a local docs website.
- Browsed the project tree, model details, and the lineage graph.
- Saw your YAML descriptions show up as rendered docs.

## Common confusion

> "Port 8080 is in use."

Add `--port 9000` (or any free port): `dbt docs serve --port 9000`.

> "The browser didn't open."

Some terminals don't open browsers automatically. Just navigate manually to
`http://localhost:8080`.

> "I see 'Failed to parse the project' or similar."

Run `dbt parse` first — if that fails, fix the parse error. `dbt docs generate`
needs a valid project.

> "The lineage graph icon is hidden."

It's a small blue circle in the bottom-right corner. Sometimes browser zoom
hides it — try Ctrl+0 to reset zoom.

> "Can I edit the docs without changing my SQL?"

The descriptions come from YAML, not SQL. Edit `_stg_models.yml`,
`_marts_models.yml`, etc. Re-run `dbt docs generate` to update the docs.

> "What's the difference between `manifest.json` and `catalog.json`?"

- `manifest.json` — your project structure (models, tests, refs). Generated
  during any dbt command.
- `catalog.json` — additional warehouse metadata (column types, row counts).
  Generated specifically by `dbt docs generate`.

Both are read by `dbt docs serve` to render the UI.

---

You've used all the basic dbt features. Two more files: a reference for errors,
and what to do next.

← Previous&nbsp; [`23d-compiled-sql.md`](23d-compiled-sql.md) &nbsp;·&nbsp; Next →&nbsp; [`25-common-errors.md`](25-common-errors.md)
