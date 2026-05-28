# 23d — Reading compiled SQL (the debugging superpower)

← Previous&nbsp; [`23c-multi-target.md`](23c-multi-target.md) &nbsp;·&nbsp; Next →&nbsp; [`23e-generate-schema-name.md`](23e-generate-schema-name.md)

---

Several earlier files mentioned "look at `target/compiled/`" without dwelling. This
file dwells. **Reading compiled SQL is the single most useful debugging skill in
dbt.** Once you internalize it, weird errors stop being mysterious.

Short file. ~10 minutes. Mostly clicking around in the `target/` folder.

## What's in `target/compiled/`

After any `dbt compile`, `dbt run`, or `dbt build`, dbt writes the rendered SQL
of every model and test to `target/compiled/`. The folder structure mirrors your
project:

```
target/compiled/
└── my_first_project/
    └── models/
        ├── staging/
        │   ├── stg_customers.sql
        │   └── stg_orders.sql
        └── marts/
            └── dim_customers.sql
    └── tests/
        └── ... (custom tests)
    └── models/staging/_stg_models.yml/  (YAML-based tests)
        ├── not_null_stg_customers_customer_id.sql
        └── ...
```

Each `.sql` file is what dbt **actually sent to BigQuery**. Pure SQL. No Jinja.
No `{{ }}`. The output of dbt's compile step.

## Why this is gold

When dbt errors with "column not found" or "syntax error," BigQuery's error
message refers to the **compiled SQL line numbers**, not your model file's line
numbers. Reading the compiled file lets you map "the error said line 12" to "the
specific clause that failed."

Also: when you're not sure what `{{ ref(...) }}` or `{{ source(...) }}` is
resolving to, the compiled file shows you exactly.

## Step 1: Run a compile

```bash
dbt compile
```

This does NOT execute SQL against BigQuery — it just renders the Jinja and
writes to `target/compiled/`.

(Unlike `dbt run`, `dbt compile` is cheap and offline.)

## Step 2: Look at `dim_customers` compiled

```bash
cat target/compiled/my_first_project/models/marts/dim_customers.sql
```

You'll see something like:

```sql
with customers as (
    select * from `silver-spaceman-12345`.`dbt_tutorial`.`stg_customers`
),

orders as (
    select * from `silver-spaceman-12345`.`dbt_tutorial`.`stg_orders`
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
    ...
```

Compare to the source in `models/marts/dim_customers.sql`. The differences:

| Source (Jinja) | Compiled (SQL) |
|---|---|
| `{{ ref('stg_customers') }}` | `` `silver-spaceman-12345`.`dbt_tutorial`.`stg_customers` `` |
| `{{ ref('stg_orders') }}` | `` `silver-spaceman-12345`.`dbt_tutorial`.`stg_orders` `` |
| `{{ config(materialized='table') }}` | (gone — config is metadata, not SQL) |

That's it. Same model, with Jinja resolved. **This is the SQL BigQuery actually
sees.**

## Step 3: Look at a test compiled

Tests also compile to plain SQL. Look at one:

```bash
cat target/compiled/my_first_project/models/staging/_stg_models.yml/not_null_stg_customers_customer_id*.sql
```

(That trailing `*` glob handles a hash suffix dbt sometimes adds.)

You'll see something like:

```sql
select customer_id
from `silver-spaceman-12345`.`dbt_tutorial`.`stg_customers`
where customer_id is null
```

That's the entire test. The not_null test for stg_customers.customer_id is
exactly: "select customer_id where it's null." If this query returns rows, the
test fails.

## Step 4: Use compiled SQL to debug a real error

Let's deliberately break something. Edit `models/marts/dim_customers.sql`. Change
one column reference, say:
```sql
c.first_name,
```
to:
```sql
c.firstName,    -- WRONG — column doesn't exist
```

Save and run:

```bash
dbt run --select dim_customers
```

dbt errors with something like:

```
Database Error in model dim_customers
  Unrecognized name: firstName at [some-line]
  compiled code at target/compiled/my_first_project/models/marts/dim_customers.sql
```

Now follow the trail. Open the compiled file:

```bash
cat target/compiled/my_first_project/models/marts/dim_customers.sql
```

Search for `firstName` (line ~22 in the compiled). You'll see the typo right
where you put it. Compare to your source `dim_customers.sql` and verify it
matches.

**Two things to notice:**

1. The error mentions a line in the **compiled** file. If you look at the source
   file the error is at a different line (because Jinja expands).
2. Once you know which line has the bad column, the fix is obvious.

Revert the typo. Re-run. Things work again.

## Step 5: `target/run/` — the actual executed wrapper

There's another folder: `target/run/`. This has the **wrapped** version — the
compiled SQL with `CREATE OR REPLACE TABLE ... AS (` around it.

```bash
cat target/run/my_first_project/models/marts/dim_customers.sql
```

You'll see something like:

```sql
create or replace table `silver-spaceman-12345`.`dbt_tutorial`.`dim_customers`
options(
    description="""...""",
    labels=[...]
)
as (

  -- ... the body from compiled/ ...

);
```

`target/run/` is what dbt literally sent. `target/compiled/` is the inner body. For
debugging the LOGIC, look at compiled. For debugging the EXECUTION (was it
CREATE TABLE? CREATE VIEW?), look at run.

## Step 6: The full debugging workflow

When something goes wrong:

1. **Read the error message** from dbt/BigQuery.
2. **Find the compiled file** the error references (`target/compiled/.../X.sql`).
3. **Read the compiled SQL** — it's plain SQL.
4. **Copy the SQL into the BigQuery console** and run it directly. The error from
   BigQuery is the authoritative one.
5. Now you can iterate quickly: edit the SQL in the console, find the fix, then
   port the fix back to your model file.

That cycle — error → compiled file → BQ console → fix in source — is the
day-to-day dbt debugging loop. Once it's habit, you stop getting stuck.

## Step 7: A pro tip — read manifest.json too

`target/manifest.json` has dbt's full internal model of the project: every model,
every test, every source, every macro, every dependency. It's large (often
several MB). You won't read it casually, but `jq` makes it useful:

```bash
# What does dim_customers depend on?
jq '.nodes."model.my_first_project.dim_customers".depends_on.nodes' target/manifest.json
```

That'd show:
```json
[
  "model.my_first_project.stg_customers",
  "model.my_first_project.stg_orders"
]
```

You don't NEED this for normal dev. But when a CI tool or third-party integration
asks for "the manifest" — this is what they mean.

## What did you just do?

- Ran `dbt compile` to populate `target/compiled/`.
- Read the compiled SQL for `dim_customers` and saw exactly what BQ executes.
- Read a compiled test query and saw it's a simple `select ... where null` check.
- Introduced a column typo, traced through the error → compiled file → BQ
  console workflow.
- Learned the difference between `target/compiled/` (body) and `target/run/`
  (wrapped).

## Common confusion

> "Why do I see different line numbers in errors vs my source file?"

Because the compiled SQL is longer than your source (Jinja expanded into longer
strings). Error messages reference the compiled line. Always navigate via the
compiled file to find the offending statement.

> "Does `target/compiled/` update automatically?"

Only when you `dbt compile`, `dbt run`, or `dbt build`. If you edit a model and
don't re-run, the compiled file is stale. Re-run before reading.

> "Can I commit `target/` to git?"

No. Standard `.gitignore`. The folder is build output; it regenerates from your
source files. Committing it pollutes diffs.

> "What's `target/manifest.json` vs `target/compiled/`?"

- `manifest.json` — the structural model of the project (what depends on what).
  One file. Read by other tools.
- `target/compiled/` — the rendered SQL for each model. One file per model. Read
  by humans for debugging.

> "If the compiled SQL is what runs, can I just write SQL directly without
> Jinja?"

Yes — a model with zero `{{ }}` is valid. But you lose ref()/source()/config()
benefits. Almost every model has at least one Jinja call. Pure SQL is a
last-resort or a one-off.

---

✅ **Part D checkpoint** — you can debug like a pro. Next: one last
configuration pattern that confuses every beginner, then documentation.

← Previous&nbsp; [`23c-multi-target.md`](23c-multi-target.md) &nbsp;·&nbsp; Next →&nbsp; [`23e-generate-schema-name.md`](23e-generate-schema-name.md)
