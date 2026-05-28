# 20 — Break a test on purpose

← Previous&nbsp; [`19-running-tests.md`](19-running-tests.md) &nbsp;·&nbsp; Next →&nbsp; [`21-dbt-build.md`](21-dbt-build.md)

---

A test is only useful if it actually fails when something's wrong. Let's prove
ours work by introducing bad data, watching the test fail, then reverting.

This is a critical exercise. Skipping it leaves you with tests you can't trust.

## Step 1: Inject bad data into the seed

Open `seeds/raw_orders.csv` in your editor. Find one of the rows with `status =
completed`. Change `completed` to `lost`. Save.

For example, change:
```
4,1,2018-01-09,completed,23
```
to:
```
4,1,2018-01-09,lost,23
```

`lost` is not in our accepted-values list (`['completed', 'returned']`), so the
`accepted_values` test on `stg_orders.status` should fail.

## Step 2: Reload the seed

Seeds aren't auto-reloaded. We need to re-run `dbt seed` so the new (bad) data
lands in BigQuery:

```bash
dbt seed --select raw_orders
```

You'll see:
```
1 of 1 OK loaded seed file dbt_tutorial.raw_orders ... [INSERT 10 in X.XXs]
```

The seed table now has 1 row with `status = 'lost'`. The staging view
`stg_orders` shows that row immediately (views don't need rebuilding — they read
the table live).

## Step 3: Run tests on stg_orders

```bash
dbt test --select stg_orders
```

Expected output (key part):

```
1 of N START test ... not_null_stg_orders_order_id ........ [RUN]
2 of N START test ... unique_stg_orders_order_id ........... [RUN]
3 of N START test ... accepted_values_stg_orders_status__completed__returned [RUN]
...
3 of N FAIL 1 accepted_values_stg_orders_status__completed__returned [FAIL 1 in X.XXs]
...

Failure in test accepted_values_stg_orders_status__completed__returned (models/staging/_stg_models.yml)
  Got 1 result, configured to fail if != 0

  compiled code at target/compiled/my_first_project/models/staging/_stg_models.yml/accepted_values_stg_orders_status__completed__returned_XXXXX.sql

Done. PASS=N-1 WARN=0 ERROR=1 SKIP=0 NO-OP=0 TOTAL=N
```

The test FAILED. dbt is telling you:

- The test query returned 1 row.
- The test expects 0 rows.
- Therefore: bad data exists in your table.

This is exactly what you want — the test caught the problem.

## Step 4: See which row failed

Open the compiled test SQL that dbt referenced:

```bash
cat target/compiled/my_first_project/models/staging/_stg_models.yml/accepted_values_stg_orders_status__completed__returned*.sql
```

You'll see something like:

```sql
with all_values as (
    select
        status as value_field,
        count(*) as n_records
    from `sweet-mountain-471812`.`dbt_tutorial`.`stg_orders`
    group by status
)

select *
from all_values
where value_field not in ('completed','returned')
```

Run this query directly in BigQuery (or via `bq query`):

```bash
bq query --use_legacy_sql=false \
  "SELECT status, COUNT(*) AS n FROM \`your-project-id.dbt_tutorial.stg_orders\`
   WHERE status NOT IN ('completed','returned')
   GROUP BY status"
```

You'll see:
```
| status | n |
|--------|---|
| lost   | 1 |
```

Exactly 1 row of bad data. That's the row we injected. The test isn't lying.

## Step 5: Trust check — the OTHER tests still pass

Look at the full output more carefully. Tests that DIDN'T involve `status` should
still be passing:

- `not_null_stg_orders_order_id` — passes (we didn't break order_ids).
- `unique_stg_orders_order_id` — passes (no duplicates).
- `relationships_stg_orders_customer_id...` — passes (customer_id still valid).

Only the `accepted_values` test on `status` failed, because that's the one whose
assertion we violated.

This is what good tests look like: precise, scoped, fail exactly when the data
that backs them is wrong, otherwise pass.

## Step 6: Revert

Now fix the data:

1. Open `seeds/raw_orders.csv`.
2. Change `lost` back to `completed`.
3. Save.

Reload the seed:

```bash
dbt seed --select raw_orders
```

Re-test:

```bash
dbt test --select stg_orders
```

Expected: all tests PASS now.

```
Done. PASS=N WARN=0 ERROR=0 SKIP=0 NO-OP=0 TOTAL=N
```

## Why this exercise matters

You now know — empirically, not abstractly — that:

1. The test runs.
2. The test fails when data is bad.
3. dbt tells you which test failed.
4. You can find the specific bad rows by inspecting the compiled test SQL.
5. Fixing the data clears the failure.

That's trust. You can trust your tests because you've watched them work.

## A second break to try (optional)

For practice, break a different test:

1. Edit `raw_customers.csv`. Make TWO rows have the same `id` (e.g., change row 2 from `id=2` to `id=1`). Save.
2. `dbt seed --select raw_customers`.
3. `dbt test --select stg_customers`.
4. Watch the `unique` test fail (`FAIL 1` — 1 duplicate group).
5. Revert.

## A third break (advanced)

Try the `relationships` test:

1. Edit `raw_orders.csv`. Change a row's `customer_id` to `999` (no customer
   with id 999 exists).
2. `dbt seed --select raw_orders`.
3. `dbt test --select stg_orders`.
4. Watch the `relationships` test fail.
5. Revert.

## What did you just do?

- Injected one row of bad data into a seed.
- Re-ran the seed so the bad data landed in BigQuery.
- Watched the `accepted_values` test catch it.
- Found the specific bad row by running the test's underlying query.
- Reverted the data and watched the test go green again.

You have now seen, with your own eyes, the test catch a problem. That's the bar
for "trustworthy tests."

## Common confusion

> "Why didn't the staging view break too?"

It didn't. The staging view said "show me the columns from raw_orders." `lost` is
a valid string, so the view still produces it. The TEST is what flagged
`lost` as unacceptable. The model would happily pass through any garbage.

> "The output says `compiled code at ...` — what's there?"

dbt wrote the rendered SQL of the failing test to that path. Opening it shows you
exactly what query dbt ran. Useful for debugging — you can run that SQL directly
in BigQuery to see the failing rows.

> "Can I see the failing rows directly without crafting a query myself?"

Yes — `dbt test --store-failures`. dbt creates a table with the failing rows
(named `dbt_test__audit.<test_name>`). Query it after a failure:

```sql
SELECT * FROM `your-project-id.dbt_test__audit.accepted_values_stg_orders_status_...`
```

Very handy in CI: even if humans never see the run output, the failure rows are
stored for inspection.

> "What if I want some tests to warn instead of fail the whole build?"

Add `config: { severity: warn }` to the test. A warning shows in dbt output but
exits 0 (doesn't fail CI). Useful for "I want to know about this, but not block
on it."

---

You've trusted your tests. Time for the one command that ties everything
together: `dbt build`.

← Previous&nbsp; [`19-running-tests.md`](19-running-tests.md) &nbsp;·&nbsp; Next →&nbsp; [`21-dbt-build.md`](21-dbt-build.md)
