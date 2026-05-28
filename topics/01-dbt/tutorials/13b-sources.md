# 13b — Sources: the production pattern

← Previous&nbsp; [`13-first-staging-model.md`](13-first-staging-model.md) &nbsp;·&nbsp; Next →&nbsp; [`14-layered-architecture.md`](14-layered-architecture.md)

---

You've been using seeds + `{{ ref('raw_customers') }}` for raw data. That works for
this tutorial, but it's not how real projects do it. **Real projects use sources.**

This file teaches you the sources pattern by example. After this, you'll have used
both patterns and understand when each applies.

## Recap: seeds vs sources

| | Seeds | Sources |
|---|---|---|
| **Who loads the data?** | dbt (`dbt seed`) | Something else (ingestion tool, manual upload, script) |
| **Format?** | CSV in `seeds/` folder, in your git repo | A table that already exists in the warehouse |
| **How to reference?** | `{{ ref('seed_name') }}` | `{{ source('group', 'table') }}` |
| **Declared in YAML?** | Optional (for type overrides) | **Required** (`_sources.yml`) |
| **Good for?** | Small static reference data (country codes, status lookups) | Real ingested data (customer events, sales, logs) |

When you read spotify-pipeline's `dbt/` folder, you'll see `{{ source('spotify_raw',
'top_tracks') }}` everywhere. That points at a table the **Cloud Function**
created — dbt didn't load it. dbt just transforms what's already there.

## The mental shift

With seeds: dbt owns end-to-end (loads CSV → tables → models on top).

With sources: dbt only owns the models on top. The raw table got there some other
way (Fivetran, Airbyte, your own Python script, a manual `bq load`). dbt's job is to
**declare** "I know this table exists, here's where to find it, here's what columns
are in it, and here are some tests on the raw data."

Sources let you:
- Build models on data dbt didn't create.
- **Test the raw data** before transforming (catch ingestion bugs early).
- Track **source freshness** ("when was this last updated? if it's >6h, alert").
- Decouple ingestion teams from analytics teams.

## Step 1: Simulate "ingestion" with a manual BQ upload

We'll pretend our ingestion pipeline dropped a new raw table into BigQuery.

In a real project this would be Fivetran or a Python script. For this tutorial,
we'll do it by hand in the BigQuery console.

### Create a new dataset for raw data

In a real Level-3 setup, raw data and analytics live in **different datasets**
(sometimes different projects). Let's mimic that.

In your terminal:

```bash
bq mk --dataset --location=US raw_simulated
```

(`bq` came with the gcloud SDK you installed earlier.) You should see:

```
Dataset 'your-project-id:raw_simulated' successfully created.
```

Refresh the BigQuery console — you'll see a new dataset `raw_simulated` next to
your existing `dbt_tutorial`.

### Upload a CSV directly to BigQuery

We'll load `raw_orders.csv` into this dataset as `raw_orders_v2`, bypassing dbt
entirely:

```bash
bq load \
  --source_format=CSV \
  --autodetect \
  --skip_leading_rows=1 \
  raw_simulated.raw_orders_v2 \
  seeds/raw_orders.csv
```

Expected output:

```
Upload complete.
Waiting on bqjob_... ... (3s) Current status: DONE
```

In BigQuery, you'll now see `raw_simulated.raw_orders_v2` with 10 rows. dbt did NOT
do this — `bq load` did. From dbt's perspective, this table just **appeared** in
the warehouse.

## Step 2: Declare the source in `_sources.yml`

For dbt to know about a table it didn't build, you tell it via YAML.

Create `models/staging/_sources.yml`:

```bash
cat > models/staging/_sources.yml <<'EOF'
version: 2

sources:
  - name: raw_simulated
    database: "{{ target.project }}"      # your GCP project (set by dbt)
    schema: raw_simulated                  # the BigQuery dataset
    description: "Simulated ingestion landing zone."
    tables:
      - name: raw_orders_v2
        description: "Raw orders, loaded by bq load (simulating ingestion)."
        columns:
          - name: id
            description: "Order id."
            data_tests:
              - not_null
              - unique
          - name: customer_id
            data_tests:
              - not_null
          - name: status
            data_tests:
              - not_null
          - name: amount
            data_tests:
              - not_null
EOF
```

Let's read this.

### `sources:` block

The top-level key for source declarations. A list — you can declare multiple
source groups (e.g., one per upstream system).

### Each source entry

```yaml
- name: raw_simulated
  database: "{{ target.project }}"
  schema: raw_simulated
  tables:
    - name: raw_orders_v2
```

- `name:` — a **logical name** for this group of related tables. This is how
  models will refer to it (`{{ source('raw_simulated', 'raw_orders_v2') }}`).
- `database:` — for BigQuery, this is the **GCP project**. Using `{{ target.project
  }}` makes it env-aware (dev points at dev project, prod at prod). For our
  single-target tutorial, it resolves to whatever's in `profiles.yml`.
- `schema:` — for BigQuery, this is the **dataset**. We point at the new
  `raw_simulated` dataset we just created.
- `tables:` — a list of tables in that source group.

### Source-level tests

Note the `data_tests:` blocks on columns. **Source tests run against the raw
data**, before any transformation. They catch ingestion bugs at the source layer
— before they pollute downstream.

This is one of the killer features of sources: testing the raw input. With seeds
you can't really do this because seeds ARE the raw data dbt creates (any bug in
the seed CSV is something you put there yourself; ingestion bugs are different
animals).

## Step 3: Write a staging model that uses `source()`

Create a new staging model that points at the source:

```bash
cat > models/staging/stg_orders_v2.sql <<'EOF'
with src as (
    select * from {{ source('raw_simulated', 'raw_orders_v2') }}
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

The model is almost identical to `stg_orders.sql` — the only difference is
**`{{ source(...) }}`** instead of **`{{ ref(...) }}`**. Same shape, different
function.

When dbt compiles this, it looks up `raw_simulated.raw_orders_v2` in the source
declaration, finds it points at `your-project.raw_simulated.raw_orders_v2`, and
substitutes that into the SQL.

## Step 4: Build everything

```bash
dbt build
```

Expected output (paraphrased):

```
Found 4 models, 1 source, ...

... raw_customers (seed) ...
... raw_orders (seed) ...
... stg_customers (view) ...
... stg_orders (view) ...
... stg_orders_v2 (view) ...           ← new!
... source tests on raw_orders_v2 ...   ← new!
... not_null_source_raw_simulated_raw_orders_v2_id ... PASS
... unique_source_raw_simulated_raw_orders_v2_id ... PASS
... model tests ...
... dim_customers (table) ...
... mart tests ...

Done. PASS=N WARN=0 ERROR=0 SKIP=0 TOTAL=N
```

`stg_orders_v2` builds as a view, same as `stg_orders`. **And the source-level
tests ran** — dbt tested the raw `raw_orders_v2` table directly before any
staging model used it.

## Step 5: Source freshness (optional but powerful)

Once you have sources, dbt can check **how fresh they are**. In a real project,
you'd add a `freshness:` block:

```yaml
- name: raw_orders_v2
  loaded_at_field: order_date         # which column tracks "when was this loaded"
  freshness:
    warn_after: { count: 24, period: hour }   # warn if >24h old
    error_after: { count: 48, period: hour }  # fail if >48h old
```

Then `dbt source freshness` runs a query: "what's the max `order_date`? if >48h
ago, error." This catches **broken ingestion** — if your Fivetran job dies, the
source table stops updating, and `dbt source freshness` will fail before any
analyst sees stale data.

We won't add this to the tutorial (our static `order_date` would always be "very
stale"), but know it exists. It's gold for production.

## Step 6: Comparison — same data, two patterns

You now have TWO staging models that produce equivalent output:

- `stg_orders` — uses `{{ ref('raw_orders') }}`, points at a seed
- `stg_orders_v2` — uses `{{ source('raw_simulated', 'raw_orders_v2') }}`, points
  at a manually-loaded table

Query both in BigQuery — same 10 rows, same columns.

In a real project you'd never have both. You'd pick one pattern for your data
flow. **Sources are the production pattern.** Seeds are for small static
reference data only.

## Cleanup

You can delete `stg_orders_v2.sql` and remove the `_sources.yml` file if you want
to keep your project clean for the rest of the tutorial. The remaining files
(11–16) all use seeds + `ref()`.

```bash
rm models/staging/stg_orders_v2.sql models/staging/_sources.yml
```

Or leave them in — both work.

## What did you just do?

- Created a separate BigQuery dataset (simulating ingestion target).
- Manually loaded a CSV into it with `bq load` (simulating ingestion).
- Declared the table as a dbt source in `_sources.yml`.
- Added source-level tests (tests on the RAW data).
- Wrote a staging model using `{{ source(...) }}`.
- Saw the source tests pass when you ran `dbt build`.
- Learned what `dbt source freshness` is for, even if you didn't try it.

## Common confusion

> "Can a model use both `ref()` and `source()`?"

Yes. Most real staging models look like:
```sql
select * from {{ source('raw', 'orders') }}
```
And most mart models look like:
```sql
select c.*, o.amount
from {{ ref('stg_customers') }} c
left join {{ ref('stg_orders') }} o ...
```
Staging uses sources to read raw data; marts use refs to read other dbt models.

> "Why have sources if I can just hardcode the table name?"

Three reasons:
1. **Environment portability** — `{{ source(...) }}` lets `database:` and `schema:` come from env vars or `target.project`.
2. **Tests on raw data** — you can't easily put tests on a hardcoded table reference; sources are the registration that makes that possible.
3. **Documentation + DAG visibility** — dbt knows your models depend on these tables; they show up in `dbt docs` lineage graphs.

> "What's the difference between `database:` and `schema:` in BigQuery vs Postgres?"

BigQuery uses **project** + **dataset** as its two-level namespace. dbt's source
config calls them `database:` (= project) and `schema:` (= dataset) because dbt
generalizes across warehouses where they ARE called database and schema. The
mapping is:

| dbt term | BigQuery | Postgres / Snowflake |
|---|---|---|
| `database:` | project | database |
| `schema:` | dataset | schema |

> "What goes in `_sources.yml` vs `_stg_models.yml`?"

- `_sources.yml` — declarations of tables that ALREADY EXIST (not built by dbt) + tests on them.
- `_stg_models.yml` — descriptions and tests for the staging models YOU built.

Some projects combine them into one file, some keep separate. Convention varies; consistency within a project matters more than which you pick.

> "Should I use sources or seeds for THIS tutorial's data?"

For learning, seeds are fine — they were the right choice for files 11–13. Now
you've seen sources too, you can choose either. spotify-pipeline uses sources for
everything because the data comes from a real ingestion pipeline (the Cloud
Function).

---

You've used both ref-to-seed and source-to-table patterns. The rest of the course
uses the seed pattern for simplicity, but you now know which one production uses.

← Previous&nbsp; [`13-first-staging-model.md`](13-first-staging-model.md) &nbsp;·&nbsp; Next →&nbsp; [`14-layered-architecture.md`](14-layered-architecture.md)
