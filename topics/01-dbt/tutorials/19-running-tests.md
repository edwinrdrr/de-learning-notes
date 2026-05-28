# 19 — Running tests with `dbt test`

← Previous&nbsp; [`18b-custom-tests.md`](18b-custom-tests.md) &nbsp;·&nbsp; Next →&nbsp; [`20-breaking-tests.md`](20-breaking-tests.md)

---

Quick file. You've declared tests; now run them.

## Step 1: Run all tests

```bash
dbt test
```

Expected output (paraphrased; counts may vary based on how dbt enumerates):

```
Running with dbt=1.11.x
Found 3 models, 17 data tests, ...

1 of 17 START test not_null_stg_customers_customer_id ............... [RUN]
2 of 17 START test unique_stg_customers_customer_id .................. [RUN]
...
17 of 17 START test not_null_dim_customers_total_spent ............... [RUN]
1 of 17 PASS not_null_stg_customers_customer_id ..................... [PASS in X.XXs]
2 of 17 PASS unique_stg_customers_customer_id ........................ [PASS in X.XXs]
...
17 of 17 PASS not_null_dim_customers_total_spent ..................... [PASS in X.XXs]

Finished running 17 data tests in X.XXs.

Completed successfully

Done. PASS=17 WARN=0 ERROR=0 SKIP=0 NO-OP=0 TOTAL=17
```

`PASS=17` means all 17 tests passed. Your data is currently healthy.

## Step 2: Selectors work for tests too

You can run tests for specific models:

```bash
# Just tests on stg_customers
dbt test --select stg_customers

# Tests on dim_customers AND its upstream models
dbt test --select +dim_customers

# Tests in the staging folder
dbt test --select staging
```

`--select` selectors work the same way for `dbt test` as for `dbt run`. The DAG
applies to test selection too.

## Step 3: What dbt actually did

For each test, dbt:

1. Generated a SQL query (the test query).
2. Sent it to BigQuery.
3. Counted the result rows.
4. If 0 rows: PASS. If >0 rows: FAIL.

You can see the generated SQL in `target/compiled/.../tests/`:

```bash
ls target/compiled/my_first_project/tests/
```

You'll see directories with files like
`not_null_stg_customers_customer_id.sql`. Open one:

```bash
cat target/compiled/my_first_project/tests/not_null_stg_customers_customer_id.sql
```

You'll see something like:

```sql
select customer_id
from `sweet-mountain-471812`.`dbt_tutorial`.`stg_customers`
where customer_id is null
```

That's the test. If the query returns rows (any rows), customers exist with NULL
customer_id, and the test fails. If 0 rows: passes.

## Step 4: What "test FAIL" looks like

Right now everything passes. In the next file we'll break the data on purpose to
see what FAIL looks like — but a preview:

```
3 of 17 FAIL 1 accepted_values_stg_orders_status__completed__returned . [FAIL 1 in X.XXs]
```

`FAIL 1` means: the test query returned 1 row. There's 1 bad row in the table.
dbt also writes the failing rows to a table (or saves the query) so you can
inspect what went wrong. By default this happens via `--store-failures`.

To enable failure inspection:

```bash
dbt test --store-failures
```

After a fail, you can query the failing rows in BigQuery to see exactly which
rows tripped the test.

## What did you just do?

- Ran all 17 tests against your project's data.
- Saw all of them pass.
- Looked at the actual SQL dbt generated for one test.
- Learned that selectors work for `dbt test` just like for `dbt run`.

## Common confusion

> "I ran `dbt test` but the example model tests run too — even though I deleted
> the example folder?"

No — dbt only runs tests it can find in YAML files for existing models. If you
deleted `models/example/` along with its `schema.yml`, those tests are gone.
Check `target/manifest.json` or `dbt list --resource-type test` to see the full
list of currently-known tests.

> "How do I see the SQL for a test that hasn't been run?"

`dbt compile` writes all compiled SQL — including tests — to `target/compiled/`.
Even without running tests, you can compile and peek.

> "Tests are slow. Can I skip some?"

Yes, with `--exclude`:

```bash
dbt test --exclude tag:slow_test
```

You'd tag specific tests in the YAML with `config: { tags: [slow_test] }`. Then
exclude in CI's quick runs, include in nightly runs.

> "Can I have warnings instead of failures for some tests?"

Yes. Add `severity: warn` to the test config:

```yaml
- not_null:
    config:
      severity: warn
```

A failed warn-level test doesn't fail the build, but shows up as `WARN` in
output. Useful for "soft" assertions you want to monitor but not block on.

---

Tests pass. Time to deliberately break the data and see them fail. That's how you
trust them.

← Previous&nbsp; [`18b-custom-tests.md`](18b-custom-tests.md) &nbsp;·&nbsp; Next →&nbsp; [`20-breaking-tests.md`](20-breaking-tests.md)
