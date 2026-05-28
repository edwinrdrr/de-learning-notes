# dbt — tutorial: your first project, end to end (BigQuery)

A walkthrough you can follow without prior dbt experience. You'll set up the
infrastructure, write models, add tests, and serve docs. Self-contained — no
"now go read the official tutorial."

## What you'll have when done

- A working **BigQuery sandbox** (free, no credit card)
- A **dbt project** with `profiles.yml` configured
- **2 seeds** (raw CSVs you uploaded yourself)
- **2 staging models** (one per seed)
- **1 mart model** that joins them
- **6 tests** across the models (some you wrote, some auto-generated)
- A **lineage graph** you can browse in your browser via `dbt docs serve`
- A correct mental model of `compile → run → test`, materializations, refs, and sources

## Total time

~2-3 hours, depending on how much you stop to read.

## Prerequisites

- **Python 3.10+** on your machine (`python3 --version` to check)
- A **terminal** you're comfortable with (bash/zsh on Linux/macOS; WSL on Windows)
- A **Google account** (for the free BigQuery sandbox)
- An **editor** (VS Code, Sublime, vim — anything)

No prior dbt, no prior BigQuery, no prior cloud experience needed.

---

# Part 1 — BigQuery sandbox setup (~15 minutes)

## 1.1 Enable BigQuery sandbox

The BigQuery sandbox is free and doesn't require a credit card. You get 10 GB of
storage and 1 TB of queries per month, which is overkill for this tutorial.

1. Open your browser and go to **https://console.cloud.google.com/bigquery**.
2. Sign in with your Google account.
3. If this is your first time, you'll see a "Welcome to BigQuery" or a Terms of
   Service screen. Accept the ToS and continue.
4. Google will create a sandbox project automatically. You'll see the BigQuery
   console with a left-side panel listing projects.
5. **Look at the top-left of the page** — there's a dropdown showing your active
   project. It'll be something like `myname-bq-sandbox` or `silver-spaceman-12345`.
   **Write down this project ID exactly — you'll need it.** Quotes don't matter,
   case does.

> ⚠️ If the page asks you to "Enable billing," you don't need to. The sandbox works
> without billing for everything in this tutorial. Just close that prompt.

## 1.2 Install the `gcloud` CLI

You'll talk to BigQuery from your terminal, which needs the Google Cloud CLI.

**Linux**:
```bash
cd ~
curl -sSL -o gcloud-cli.tar.gz \
  https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-cli-linux-x86_64.tar.gz
tar -xzf gcloud-cli.tar.gz && rm gcloud-cli.tar.gz
./google-cloud-sdk/install.sh --quiet --path-update=true
# reload your shell
source ~/.bashrc    # or ~/.zshrc
```

**macOS** (Homebrew):
```bash
brew install --cask google-cloud-sdk
```

**Windows**: download the installer from
https://cloud.google.com/sdk/docs/install#windows.

Verify:
```bash
gcloud --version
# → Google Cloud SDK 5xx.x.x
```

## 1.3 Authenticate Application Default Credentials (ADC)

dbt will read these credentials to talk to BigQuery.

```bash
gcloud auth application-default login
```

A browser tab opens. Sign in with the **same Google account** you used for BigQuery.
After signing in, you can close the tab.

Set the quota project (this tells GCP which project to bill API calls to — irrelevant
for sandbox but suppresses a warning):
```bash
gcloud auth application-default set-quota-project <YOUR_PROJECT_ID>
```

Replace `<YOUR_PROJECT_ID>` with the ID from step 1.1.

Verify:
```bash
gcloud auth application-default print-access-token > /dev/null && echo "OK"
# → OK
```

---

# Part 2 — Install dbt (~10 minutes)

## 2.1 Create a working directory

Pick a path you'll remember. Anywhere is fine:
```bash
mkdir -p ~/learning/dbt-tutorial
cd ~/learning/dbt-tutorial
```

## 2.2 Set up a Python virtualenv

A virtualenv keeps dbt's dependencies isolated from your system Python.

```bash
python3 -m venv .venv
source .venv/bin/activate
# Your prompt should now show (.venv) at the start
```

> If `source` doesn't work, you're not in bash/zsh. On Windows PowerShell:
> `.venv\Scripts\Activate.ps1`.

## 2.3 Install dbt-bigquery

```bash
pip install --upgrade pip
pip install dbt-bigquery
```

This also installs `dbt-core` (the framework). Takes ~30 seconds.

Verify:
```bash
dbt --version
# Core:   1.11.x
# Plugins:
#   - bigquery: 1.9.x
```

If you see the version numbers, you're set.

---

# Part 3 — Your first dbt project (~25 minutes)

## 3.1 `dbt init`

```bash
dbt init my_first_project
```

dbt will ask you questions interactively:
- **Which database would you like to use?** → type `1` (or whatever number is next to `bigquery`) and press Enter
- **[1] oauth, [2] service_account** → type `1` for `oauth`
- **project (GCP project id)** → paste your project ID from step 1.1
- **dataset (the name of your dbt schema)** → type `dbt_tutorial`
- **threads** → type `4`
- **job_execution_timeout_seconds** → press Enter for default
- **Desired location** → type `US` (this matches BigQuery sandbox default)

When it finishes, you'll have:
```
my_first_project/
├── README.md
├── analyses/
├── dbt_project.yml
├── macros/
├── models/
│   └── example/
├── seeds/
├── snapshots/
└── tests/
```

`cd` into it:
```bash
cd my_first_project
```

## 3.2 Locate `profiles.yml`

dbt stored your connection info at `~/.dbt/profiles.yml`. Open it in your editor.

You should see something like:
```yaml
my_first_project:
  outputs:
    dev:
      type: bigquery
      method: oauth
      project: silver-spaceman-12345
      dataset: dbt_tutorial
      threads: 4
      location: US
  target: dev
```

> **Tip for later**: a real project moves `profiles.yml` into the project directory
> via `export DBT_PROFILES_DIR=$PWD` so it's repo-local. For this tutorial, leave it
> at the default `~/.dbt/profiles.yml`.

## 3.3 `dbt debug`

This is the "is everything wired up correctly" check.

```bash
dbt debug
```

Expected output ends with:
```
Connection test: [OK connection ok]
All checks passed!
```

If you see anything else, jump to **Common errors** at the bottom.

## 3.4 Run the example

dbt scaffolded two example models in `models/example/`. Let's run them so you see
what success looks like.

```bash
dbt run
```

Expected output:
```
Running with dbt=1.11.x
Found 2 models, 4 measures, 3 tests, ...
... my_first_model ... [CREATE TABLE ... in X.XXs]
... my_second_model ... [CREATE VIEW ... in X.XXs]
Done. PASS=2 WARN=0 ERROR=0 SKIP=0 TOTAL=2
```

🎉 You just ran dbt. Two tables now exist in your BigQuery project.

**Verify in BigQuery**: open https://console.cloud.google.com/bigquery, find your
project in the left panel, expand it, you'll see a `dbt_tutorial` dataset with two
tables.

## 3.5 Clean the slate

The examples are clutter. Delete them:

```bash
rm -rf models/example
```

We'll build from scratch.

---

# Part 4 — Build a real project (~45 minutes)

## 4.1 Add raw data via seeds

A **seed** in dbt is a CSV file dbt will load as a table. Real projects use seeds for
small static reference data (e.g., country codes, status lookups). For learning,
seeds are the easiest way to get raw data into BQ.

Create two CSVs:

**`seeds/raw_customers.csv`**:
```csv
id,first_name,last_name,signup_date
1,Michael,Phillips,2018-01-01
2,Shawn,Mccoy,2018-01-02
3,Kathleen,Payne,2018-01-04
4,Jimmy,Cooper,2018-01-06
5,Katherine,Rivera,2018-01-09
6,Sarah,Reed,2018-01-09
7,Martin,Long,2018-01-11
8,Frank,Owens,2018-01-12
9,Roger,Sanchez,2018-01-15
10,Annette,Phillips,2018-01-18
```

**`seeds/raw_orders.csv`**:
```csv
id,customer_id,order_date,status,amount
1,1,2018-01-01,returned,10
2,3,2018-01-02,completed,20
3,3,2018-01-04,returned,16
4,1,2018-01-09,completed,23
5,2,2018-01-11,returned,12
6,9,2018-01-12,returned,16
7,5,2018-01-14,completed,17
8,1,2018-01-18,completed,4
9,7,2018-01-19,completed,5
10,1,2018-01-21,completed,12
```

(These are scaled-down "Jaffle Shop" data — dbt Labs's canonical demo.)

## 4.2 `dbt seed`

```bash
dbt seed
```

dbt creates two tables in BigQuery: `raw_customers` and `raw_orders` in your
`dbt_tutorial` dataset. Output:
```
... raw_customers ... [SUCCESS]
... raw_orders ... [SUCCESS]
Done. PASS=2 WARN=0 ERROR=0 SKIP=0 TOTAL=2
```

> Seeds are technically a hack for static reference data; in real projects, raw
> tables come from an ingestion pipeline (Airflow, Fivetran, your own Python script).
> But for learning, seeds get you to "I have raw data in the warehouse" in 30 seconds.

## 4.3 Your first staging model

The convention: **one staging model per raw source table**. Staging models do light
cleanup — rename columns, cast types, drop irrelevant fields. They're materialized
as **views** (cheap; the underlying table is already in the warehouse).

Create the staging folder:
```bash
mkdir -p models/staging
```

**`models/staging/stg_customers.sql`**:
```sql
with src as (
    select * from {{ ref('raw_customers') }}
)
select
    id as customer_id,
    first_name,
    last_name,
    signup_date
from src
```

**`models/staging/stg_orders.sql`**:
```sql
with src as (
    select * from {{ ref('raw_orders') }}
)
select
    id as order_id,
    customer_id,
    order_date,
    status,
    amount
from src
```

Note the `{{ ref('raw_customers') }}` syntax — this tells dbt "the table built by the
model (or seed) named `raw_customers`." dbt resolves it to the fully-qualified BQ
name (`silver-spaceman-12345.dbt_tutorial.raw_customers`) at compile time.

Run them:
```bash
dbt run --select staging
```

Output: 2 models, both as views.

**Verify**: in the BigQuery console, your `dbt_tutorial` dataset now has
`stg_customers` and `stg_orders` views.

## 4.4 Your first mart

Marts are the analytics-ready output. They typically have a clear grain (one row per
customer; one row per order; one row per customer-day).

Create the marts folder:
```bash
mkdir -p models/marts
```

**`models/marts/dim_customers.sql`** — one row per customer, with stats:
```sql
with customers as (
    select * from {{ ref('stg_customers') }}
),

orders as (
    select * from {{ ref('stg_orders') }}
),

customer_orders as (
    select
        customer_id,
        min(order_date) as first_order_date,
        max(order_date) as most_recent_order_date,
        count(*) as orders_count,
        sum(amount) as total_spent
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
    coalesce(co.orders_count, 0) as orders_count,
    coalesce(co.total_spent, 0) as total_spent
from customers c
left join customer_orders co using (customer_id)
```

By default this materializes as a `view`. Marts are usually `table`s (faster downstream).
Override at the top of the file:

```sql
{{ config(materialized='table') }}

with customers as (
    -- ... rest of the SQL unchanged ...
```

Run only the marts:
```bash
dbt run --select marts
```

Now you have `dim_customers` as a table.

**Verify the data**:
```bash
# In BigQuery console, run:
SELECT * FROM `<YOUR_PROJECT_ID>.dbt_tutorial.dim_customers`
ORDER BY total_spent DESC LIMIT 5
```

You should see customer 1 (Michael Phillips) at the top with $39 across 3 completed orders.

## 4.5 See the DAG

Run everything:
```bash
dbt run
```

Output shows the order dbt picked:
```
... 1 of 5 START sql ... seed raw_customers
... 2 of 5 START sql ... seed raw_orders
... 3 of 5 START sql view stg_customers
... 4 of 5 START sql view stg_orders
... 5 of 5 START sql table dim_customers
```

dbt figured out the dependencies from `{{ ref(...) }}` calls and ran in topological
order. **You did not tell dbt the order. That's the whole point.**

---

# Part 5 — Tests (~20 minutes)

Tests are how you sleep at night.

## 5.1 Add column tests

dbt has four built-in column tests: `not_null`, `unique`, `accepted_values`,
`relationships`. Add them via a `.yml` file.

**`models/staging/_stg_models.yml`**:
```yaml
version: 2

models:
  - name: stg_customers
    columns:
      - name: customer_id
        tests:
          - not_null
          - unique
  - name: stg_orders
    columns:
      - name: order_id
        tests:
          - not_null
          - unique
      - name: status
        tests:
          - accepted_values:
              arguments:
                values: ['completed', 'returned']
      - name: customer_id
        tests:
          - relationships:
              arguments:
                to: ref('stg_customers')
                field: customer_id
```

> Note the `arguments:` nesting under `accepted_values` and `relationships`. dbt
> 1.10+ deprecated the top-level `values:` form. The cheatsheet has more.

## 5.2 Run the tests

```bash
dbt test
```

Expected:
```
... 6 of 6 PASS unique_stg_orders_order_id ... [PASS in ...]
Done. PASS=6 WARN=0 ERROR=0 SKIP=0 TOTAL=6
```

6 tests: 2 on `stg_customers` (not_null + unique on customer_id), 4 on `stg_orders`
(not_null + unique on order_id, accepted_values on status, relationships on
customer_id).

## 5.3 Break something on purpose

A test is only useful if it fails when it should. Edit `seeds/raw_orders.csv` and
change one `status` value from `completed` to `lost`. Then:

```bash
dbt seed                # reload the seed with the bad value
dbt test --select stg_orders
```

You should see:
```
... FAIL 1 accepted_values_stg_orders_status__completed__returned ...
Done. PASS=3 WARN=0 ERROR=1 SKIP=0 TOTAL=4
```

The test caught the bad data. **This is the entire point of dbt tests.** Revert the
CSV, re-run `dbt seed`, confirm tests pass again.

---

# Part 6 — dbt build + materializations (~15 minutes)

## 6.1 `dbt build` is the one command you'll use most

```bash
dbt build
```

`build` is `seed + run + test` in one, in dependency order. If any test fails, the
downstream models are skipped (you don't propagate bad data).

This is what CI runs.

## 6.2 Materialization deep-dive

Open `dbt_project.yml`. Near the bottom:
```yaml
models:
  my_first_project:
    +materialized: view    # default
```

Change to:
```yaml
models:
  my_first_project:
    staging:
      +materialized: view
    marts:
      +materialized: table
```

Now `dbt run` will rebuild your staging as views (lazy SELECT) and marts as tables
(physical, fast to query). The `+` prefix means "this applies to everything in the
folder, but individual models can override."

Run:
```bash
dbt run
```

Notice marts say "CREATE TABLE" while staging says "CREATE VIEW." That's the
materialization in action.

---

# Part 7 — Documentation (~5 minutes)

```bash
dbt docs generate    # builds a JSON of all your models
dbt docs serve       # spins up a local web server
```

A browser tab opens at `http://localhost:8080`. Click around:
- **Project** view: tree of all models
- **Database** view: tree of all tables/views in BQ (your dbt-created ones)
- Click any model to see its description, columns, tests, lineage graph
- Click the **blue graph icon at the bottom-right** to see the full DAG

Ctrl+C in the terminal to stop the server.

---

# Common errors (read this when stuck)

| Error | Cause | Fix |
|---|---|---|
| `dbt debug: Connection test: ... DBT_BIGQUERY_PROJECT not set` | Wrong project ID in `profiles.yml` | Re-edit `~/.dbt/profiles.yml`, paste the exact project ID from the BQ console top-left |
| `Permission denied: bigquery.jobs.create` | Your Google account isn't authenticated | `gcloud auth application-default login` again |
| `Not found: Dataset ... was not found in location US` | Dataset doesn't exist yet AND your `location` doesn't match BQ's default | In `profiles.yml`, set `location: US`. dbt will create the dataset on first run. |
| `Compilation Error in model … Depends on a node named raw_customers which was not found` | You ran `dbt run` before `dbt seed` | `dbt seed` first, then `dbt run`. Or use `dbt build` which does both in order. |
| `OperationalError ... User does not have permission ... project: foo` | Quota project not set on ADC | `gcloud auth application-default set-quota-project <YOUR_PROJECT_ID>` |
| `accepted_values: ... Validation Error` | Old `values:` syntax | Wrap in `arguments:` block per the YAML above |
| `dbt run` deletes my BigQuery tables | You ran `dbt run --full-refresh` on an incremental model | Incremental defaults to append-only; only `--full-refresh` drops + recreates |
| `Profile my_first_project not found` | `DBT_PROFILES_DIR` is set to wrong path, or profiles.yml is missing | `ls ~/.dbt/profiles.yml` — if missing, re-run `dbt init` |

---

# What's next

You now know:
- ✅ Setting up dbt against a real cloud warehouse
- ✅ Seeds, sources, models, refs, tests
- ✅ Materializations (view vs table)
- ✅ `dbt build` workflow
- ✅ dbt docs

**Recommended next steps**:

1. **Re-read [`concepts.md`](concepts.md)** in this folder. Things that were abstract
   in the first read will land now that you've used them.

2. **Read [`cheatsheet.md`](cheatsheet.md)**. Skim the parts you haven't used yet —
   `--select` selectors, incremental syntax, `--defer` for Slim CI.

3. **Read [`applied-in-spotify-pipeline.md`](applied-in-spotify-pipeline.md)**. Now
   that you've built a dbt project, walking through how spotify-pipeline does it will
   make sense — same patterns, slightly more polished.

4. **Add an incremental model** to your tutorial project. Create an
   `orders_with_running_total.sql` mart that uses
   `materialized='incremental'` with `is_incremental()` to only add new orders. The
   docs have a good example.

5. **Try `dbt build --select state:modified.body+ --defer --state ./path`** with a
   committed manifest somewhere. This is Slim CI; the spotify-pipeline doc 09 walks
   through it.

6. **Build something of your own** — pick a dataset you care about (your GitHub
   activity, your Spotify history, a CSV from a hobby project) and model it. The
   leap from "I followed a tutorial" to "I can use dbt" happens here.

Tutorial wrapped. You should now be able to read spotify-pipeline's `dbt/` folder and
understand most of what's going on.
