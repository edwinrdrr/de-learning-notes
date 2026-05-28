# 22b — Building an incremental model for real

← Previous&nbsp; [`22-materializations.md`](22-materializations.md) &nbsp;·&nbsp; Next →&nbsp; [`23-configuring-mat.md`](23-configuring-mat.md)

---

File 22 explained incremental in theory. This file actually builds one and watches
the only-new-rows behavior happen. Incremental is the materialization you'll trip
on most in real projects (every gotcha you hit will be one of the three things in
this file), so it's worth doing for real.

~20 minutes. There's some typing and waiting between dbt runs.

## What we're going to build

A new mart, `fct_orders_incremental`, that materializes as incremental on top of
your existing `stg_orders`. Then we'll:

1. Build it the first time — it builds as a full table.
2. Add new rows to the raw seed and re-run — only the new rows get inserted.
3. Add a new column to the model — see `on_schema_change` matter.
4. Run `--full-refresh` — see the table get rebuilt from scratch.

By the end you'll have hit all three things that confuse people about incremental.

## Step 1: Make sure `stg_orders` has `order_date` available

Check `models/staging/stg_orders.sql`. It should select `order_date` (it was in
the original seed). If you've been following along, it does.

We'll use `order_date` as the "watermark" — the column dbt checks to decide what's
new.

## Step 2: Create the incremental model

Create `models/marts/fct_orders_incremental.sql`:

```sql
{{ config(
    materialized='incremental',
    unique_key='order_id'
) }}

select
    order_id,
    customer_id,
    order_date,
    status,
    amount
from {{ ref('stg_orders') }}

{% if is_incremental() %}
  where order_date > (select max(order_date) from {{ this }})
{% endif %}
```

Two new things:

- **`{{ config(materialized='incremental', unique_key='order_id') }}`** — tells
  dbt this is incremental. `unique_key` is the column dbt uses to detect "is this
  row already in the table" — important for the MERGE behavior.
- **`{% if is_incremental() %} ... {% endif %}`** — Jinja conditional. On the
  *first* run (table doesn't exist yet), `is_incremental()` returns false; the
  WHERE clause is skipped; all rows get loaded. On *subsequent* runs (table
  exists), it returns true; the WHERE clause kicks in; only rows newer than the
  current max get loaded.

## Step 3: First build — full load

```bash
dbt build --select fct_orders_incremental
```

Expected:

```
1 of 1 START sql incremental model dbt_tutorial.fct_orders_incremental . [RUN]
1 of 1 OK created sql incremental model dbt_tutorial.fct_orders_incremental [CREATE TABLE (5.0 rows, ...)]
```

Note: **CREATE TABLE**, not MERGE. First build always rebuilds.

Verify in BigQuery (console or `bq query`):

```sql
select count(*) from `your-project.dbt_tutorial.fct_orders_incremental`;
```

You'll get 5 (or however many orders were in your seed). The table is fully
populated.

## Step 4: Look at what dbt actually ran

Check the compiled output:

```bash
cat target/compiled/my_first_project/models/marts/fct_orders_incremental.sql
```

You'll see the WHERE clause IS NOT THERE — because `is_incremental()` returned
false. The SELECT was unconditional.

Then look at `target/run/`:

```bash
cat target/run/my_first_project/models/marts/fct_orders_incremental.sql
```

You'll see the wrapping `create or replace table ... as (...)`. First run = CREATE.

## Step 5: Add new rows to the source

Open `seeds/raw_orders.csv` and append a few new rows with **later dates** than
anything in the file:

```csv
6,2,2026-06-15,completed,50.00
7,3,2026-06-20,completed,75.50
```

Save.

## Step 6: Re-run — incremental kicks in

```bash
dbt build --select +fct_orders_incremental
```

The `+` selector pulls in upstream models (seeds, staging). You'll see:

```
1 of 4 START seed file dbt_tutorial.raw_orders ............ [RUN]
1 of 4 OK loaded seed file dbt_tutorial.raw_orders ........ [INSERT 7 rows]
2 of 4 START sql view model dbt_tutorial.stg_orders ....... [RUN]
2 of 4 OK created sql view model dbt_tutorial.stg_orders .. [CREATE VIEW]
3 of 4 START sql incremental model dbt_tutorial.fct_orders_incremental . [RUN]
3 of 4 OK created sql incremental model dbt_tutorial.fct_orders_incremental [MERGE (2.0 rows, ...)]
```

**Notice the difference:** `MERGE (2.0 rows, ...)`. Not CREATE TABLE. Not even
INSERT 7 rows. Just 2 rows. **Only the new ones.**

This is the win. The other 5 rows didn't get touched.

Look at the compiled SQL again:

```bash
cat target/compiled/my_first_project/models/marts/fct_orders_incremental.sql
```

Now you'll see the WHERE clause **is** there:

```sql
where order_date > (select max(order_date) from `...`.`fct_orders_incremental`)
```

Look at `target/run/` for the actual MERGE statement BigQuery executed:

```bash
cat target/run/my_first_project/models/marts/fct_orders_incremental.sql
```

You'll see a `merge into ... using (...) on (...) when matched then ... when not matched then insert`.
That's the dbt-generated MERGE — which uses your `unique_key` to decide what's
"matched" (update) vs "not matched" (insert).

Verify count: 7 total rows now.

## Step 7: The `on_schema_change` gotcha

Real projects evolve. Someone adds a column to the source. Now what?

Edit `seeds/raw_orders.csv` and add a new column called `notes`. Update header and
all rows:

```csv
id,user_id,order_date,status,amount,notes
1,1,2026-01-01,returned,10.99,
2,3,2026-01-04,completed,15.00,
3,3,2026-01-07,completed,20.00,
4,1,2026-01-08,completed,12.50,
5,1,2026-01-10,completed,8.00,
6,2,2026-06-15,completed,50.00,vip
7,3,2026-06-20,completed,75.50,priority
```

Update `models/marts/fct_orders_incremental.sql` to select the new column:

```sql
{{ config(
    materialized='incremental',
    unique_key='order_id'
) }}

select
    order_id,
    customer_id,
    order_date,
    status,
    amount,
    notes
from {{ ref('stg_orders') }}

{% if is_incremental() %}
  where order_date > (select max(order_date) from {{ this }})
{% endif %}
```

You'll also need to add `notes` to `stg_orders.sql`. Just add a `notes` column to
the select.

Now run:

```bash
dbt build --select +fct_orders_incremental
```

This will likely fail with something like:

```
Database Error
  Column notes does not exist in table fct_orders_incremental
```

Or — depending on your dbt version — it might succeed but **silently ignore the
new column**. Either way, this is the `on_schema_change` problem.

Fix it. Update the config:

```sql
{{ config(
    materialized='incremental',
    unique_key='order_id',
    on_schema_change='append_new_columns'
) }}
```

Re-run. Now dbt issues an `ALTER TABLE ... ADD COLUMN notes` first, then runs the
MERGE.

The four `on_schema_change` options:

- **`ignore`** (default) — new columns in the SELECT are silently dropped on
  insert. You won't see new column data. Most common source of "where did my data
  go" confusion.
- **`append_new_columns`** — dbt ALTERs the table to add new columns. **What you
  usually want.**
- **`sync_all_columns`** — dbt ALTERs to add new AND drop columns no longer in the
  SELECT. Aggressive; use with care.
- **`fail`** — error out if the schema doesn't match. Safer for production; forces
  you to think.

## Step 8: `--full-refresh` — rebuild from scratch

Sometimes you need to rebuild the whole incremental table. Maybe you changed the
SELECT logic and old rows are now wrong. Maybe you changed the watermark column.
Maybe you just want a clean slate.

```bash
dbt build --select fct_orders_incremental --full-refresh
```

Watch the output — instead of `MERGE`, you'll see `CREATE TABLE` again. Same as
the first run. All 7 rows rebuilt from scratch.

**When to use `--full-refresh`:**

- After changing the SELECT logic.
- After changing `unique_key`.
- After fixing data quality issues in the source.
- In CI, when you want a deterministic baseline.

**When NOT to:**

- During normal scheduled runs. Defeats the entire point of incremental.

## Step 9: A real-project incremental pattern

In a real project, you'd combine incremental with a **lookback window** to handle
"late-arriving data" — rows that show up in the source after the watermark moved
past their natural timestamp.

```sql
{% if is_incremental() %}
  where order_date >= date_sub(
    (select max(order_date) from {{ this }}),
    interval 3 day
  )
{% endif %}
```

This rebuilds the last 3 days every run. Together with `unique_key`, the MERGE
handles late arrivals: rows with order_ids already in the table get updated; new
rows get inserted. Three days of light reprocessing every run, no missed rows.

This pattern is everywhere in production dbt. You don't need to use it today, but
when you do, the muscle memory from this file kicks in.

## What did you just do?

- Built an incremental model that uses `is_incremental()` correctly.
- Watched the first run be a CREATE TABLE.
- Watched the second run be a MERGE that only touched new rows.
- Hit the `on_schema_change` gotcha and fixed it with `append_new_columns`.
- Used `--full-refresh` to do a clean rebuild.
- Saw the lookback-window pattern that real projects use.

## Common confusion

> "Why does dbt run a MERGE instead of an INSERT?"

Because of `unique_key`. With `unique_key`, dbt has to handle the case where a
row with that ID already exists — should it insert (duplicate!) or update? MERGE
handles both in one statement. If you omit `unique_key`, dbt would do a plain
INSERT, but you'd risk duplicates.

> "What if I don't have a timestamp column?"

You need *some* way to filter "what's new." If you don't have a timestamp, you
might filter on an ID column (`where id > (select max(id) from {{ this }})`), or
a load_date you add at ingestion time. If you genuinely have no way to identify
new rows, incremental is the wrong materialization.

> "The first run failed because `{{ this }}` doesn't exist yet."

`{{ this }}` is safe inside `{% if is_incremental() %}` because the conditional
is false on the first run — the `select max(...)` never gets emitted. Don't put
`{{ this }}` outside the conditional.

> "Should I always use incremental?"

No. Incremental is complexity. Use it when full rebuilds are too slow (minutes
to hours) AND the table is mostly append-only. Tables under a million rows or
under a few seconds of build time should stay as `table`.

> "How does this compare to spotify-pipeline?"

spotify-pipeline doesn't use incremental — `fct_track_popularity_daily` is small
enough to rebuild fully every day. That's the right call for that project. But
once a fact table gets big, this is the pattern you reach for.

---

You can now build, evolve, and refresh an incremental model. The biggest
materialization gotcha in dbt is now muscle memory.

← Previous&nbsp; [`22-materializations.md`](22-materializations.md) &nbsp;·&nbsp; Next →&nbsp; [`23-configuring-mat.md`](23-configuring-mat.md)
