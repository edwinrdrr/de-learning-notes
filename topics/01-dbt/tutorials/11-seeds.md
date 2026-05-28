# 11 — Seeds: loading CSVs as raw tables

← Previous&nbsp; [`10-dbt-run.md`](10-dbt-run.md) &nbsp;·&nbsp; Next →&nbsp; [`12-jinja-and-ref.md`](12-jinja-and-ref.md)

---

dbt is for *transforming* data that's already in the warehouse. It does not move
or ingest data — that's the job of ingestion tools (Fivetran, Airbyte, your own
Python). So how do we get data into BigQuery for this tutorial without setting up
a real ingestion pipeline?

The answer: **seeds**. Seeds are dbt's hack for loading small CSVs as tables.

## What are seeds?

A seed is a `.csv` file in your project's `seeds/` folder. When you run `dbt seed`,
dbt:

1. Reads the CSV.
2. Creates a table in BigQuery with the CSV's name.
3. Loads the rows into it.

That table is then queryable like any other. Models can reference it with `{{
ref('seed_name') }}` (the same `ref()` you'd use for another model).

Seeds are intentionally *limited* — they're meant for **small static reference
data**: country codes, status code lookup tables, mapping CSVs from finance.
**Not for real data.** A seed is hand-curated and committed to git; that's not how
real fact tables work.

For learning, seeds are perfect: they get raw data into your warehouse in 30
seconds without setting up an ingestion pipeline.

## Step 1: Create two seed files

We'll simulate "customers and orders" data — a tiny version of the canonical
"Jaffle Shop" tutorial dataset that dbt Labs uses.

From inside your project folder (`my_first_project/`):

```bash
ls seeds/
# (empty)
```

Create the first seed:

```bash
cat > seeds/raw_customers.csv <<'EOF'
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
EOF
```

(If `cat > file <<EOF ... EOF` is unfamiliar, just open `seeds/raw_customers.csv`
in your editor and paste the content between the EOF lines.)

Create the second seed:

```bash
cat > seeds/raw_orders.csv <<'EOF'
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
EOF
```

Verify:

```bash
ls seeds/
# raw_customers.csv  raw_orders.csv
```

And peek at the first few lines:

```bash
head -3 seeds/raw_customers.csv
# id,first_name,last_name,signup_date
# 1,Michael,Phillips,2018-01-01
# 2,Shawn,Mccoy,2018-01-02
```

## Step 2: Run `dbt seed`

```bash
dbt seed
```

Expected output:

```
Running with dbt=1.11.x
Found 0 models, 0 measures, 0 tests, 0 snapshots, 2 sources, 0 exposures, 0 metrics, 433 macros, 0 groups, 0 semantic models, 0 saved queries

Concurrency: 4 threads (target='dev')

1 of 2 START seed file dbt_tutorial.raw_customers ............ [RUN]
2 of 2 START seed file dbt_tutorial.raw_orders ................ [RUN]
1 of 2 OK loaded seed file dbt_tutorial.raw_customers ......... [INSERT 10 in X.XXs]
2 of 2 OK loaded seed file dbt_tutorial.raw_orders ............ [INSERT 10 in X.XXs]

Finished running 2 seeds in X.XXs.
```

`INSERT 10` means BigQuery now has a table with 10 rows. Same for orders.

## Step 3: Verify in BigQuery

Open the BigQuery console (https://console.cloud.google.com/bigquery), expand your
project on the left panel. You'll see a `dbt_tutorial` dataset (dbt created it
since it didn't exist). Inside, two tables: `raw_customers` and `raw_orders`.

Click `raw_customers`. The "Preview" tab shows the 10 rows you just loaded.

You can also run a query:

```sql
SELECT * FROM `your-project-id.dbt_tutorial.raw_customers` LIMIT 5
```

(Replace `your-project-id` with yours.)

## Step 4: How dbt picked column types

You might wonder: how did dbt know `signup_date` should be a DATE column and not a
STRING? It guessed from the CSV content. For the most part this works fine; if it
gets something wrong, you can override.

To check what types dbt chose:

```sql
SELECT column_name, data_type
FROM `your-project-id.dbt_tutorial.INFORMATION_SCHEMA.COLUMNS`
WHERE table_name = 'raw_customers'
```

You'll see:

```
id           | INT64
first_name   | STRING
last_name    | STRING
signup_date  | DATE
```

dbt did the right thing. If it hadn't (e.g., if you had a column with mostly
numbers but the occasional text value), you can override the schema with a
`_seeds.yml` file. We won't need that here.

## Step 5: Seeds re-run if you change the CSV

If you edit `raw_customers.csv` and re-run `dbt seed`, dbt **drops and recreates**
the table by default. The full new contents replace the old.

This is the right default for static reference data: you change the CSV in git,
the table in BQ updates to match.

## Real-world note

In production, seeds are NOT how you load big tables. They're CSVs in git — fine
for 10 country codes, terrible for 10 million sales records. For real data, you'd
use:

- An ingestion tool (Fivetran, Airbyte) → drops raw data into BigQuery directly.
- A custom Python script (like spotify-pipeline's `snapshot.py`) → writes CSV to
  GCS, then `bq load` → BigQuery.

Either way, dbt's `source()` directive (we'll see in file 12) points at those
already-loaded raw tables, just like our staging models will point at our seeds.

**For learning purposes**: seeds simulate "raw data ingested by someone else."
That's the mental substitution to make.

## What did you just do?

- Created two CSV files in `seeds/`.
- Ran `dbt seed` to load them into BigQuery.
- Verified the tables landed.
- Learned the limits of seeds (small static reference data, not real raw data).

## Common confusion

> "Why are these called 'seeds' and not 'CSVs'?"

dbt convention from the early days. It's a horticulture metaphor — small starting
data you "plant" in the warehouse. Most people would have called them "reference
CSVs," but the name stuck.

> "Can I name my CSV files whatever I want?"

Yes, with two restrictions:
1. The filename (without `.csv`) is what BigQuery will call the table.
2. The name must be a valid SQL identifier: lowercase, alphanumeric,
   underscores. No dashes. No starting with a number.

> "What if my CSV has commas inside fields (e.g., 'Phillips, Jr.')?"

Wrap that field in double quotes:

```csv
id,first_name,last_name,signup_date
1,Michael,"Phillips, Jr.",2018-01-01
```

dbt handles standard CSV quoting.

> "Do I have to re-run `dbt seed` every time I `dbt run`?"

Not if the CSV hasn't changed. Seeds are only loaded by `dbt seed`. Once they're
in BigQuery, your models can `ref()` them indefinitely. But `dbt build` (which
we'll see in file 21) does seed + run + test in one command, so the workflow is
typically `dbt build` rather than `dbt seed` and `dbt run` separately.

---

We have raw data. Now we need to write SQL that transforms it — and to do that,
we need to learn how dbt uses Jinja and what `ref()` is.

← Previous&nbsp; [`10-dbt-run.md`](10-dbt-run.md) &nbsp;·&nbsp; Next →&nbsp; [`12-jinja-and-ref.md`](12-jinja-and-ref.md)
