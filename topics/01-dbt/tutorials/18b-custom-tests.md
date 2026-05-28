# 18b — Custom (singular) tests

← Previous&nbsp; [`18-column-tests.md`](18-column-tests.md) &nbsp;·&nbsp; Next →&nbsp; [`19-running-tests.md`](19-running-tests.md)

---

Built-in tests (`not_null`, `unique`, `accepted_values`, `relationships`) cover
the common cases. But sometimes you need an assertion the built-ins can't express
— like "no customer has negative total spend" or "fct_orders matches the raw
order count."

For that, you write a **custom test** (also called a **singular test**).

Short file. ~10 minutes.

## What is a custom test?

A custom test is just a `.sql` file in the `tests/` folder. The file contains
**one SELECT statement that should return zero rows if the data is healthy**.

That's the entire contract. Same as built-in tests: zero rows = passing; any rows
= failing.

The advantage: you write whatever SQL you want. The disadvantage: you write the
SQL yourself (vs the built-ins which dbt generates for you).

## Step 1: Write a custom test

We want to assert: **no customer has negative `total_spent`**. Our model's
`COALESCE(total_spent, 0)` should guarantee this — but a test gives us belt and
suspenders.

Create `tests/assert_dim_customers_total_spent_non_negative.sql`:

```bash
cat > tests/assert_dim_customers_total_spent_non_negative.sql <<'EOF'
-- Assert: every customer's total_spent is >= 0
select
    customer_id,
    total_spent
from {{ ref('dim_customers') }}
where total_spent < 0
EOF
```

That's it. No YAML config, no `tests:` block in another file. The file IS the
test.

Reading the SQL:
- Select customers whose `total_spent` is negative.
- If the query returns rows → those rows are violations → test FAILS.
- If the query returns 0 rows → no negative spend → test PASSES.

## Step 2: Run it

```bash
dbt test --select assert_dim_customers_total_spent_non_negative
```

Expected:

```
1 of 1 START test assert_dim_customers_total_spent_non_negative .. [RUN]
1 of 1 PASS assert_dim_customers_total_spent_non_negative ........ [PASS in X.XXs]

Done. PASS=1 WARN=0 ERROR=0 SKIP=0 NO-OP=0 TOTAL=1
```

It passes because no customer in your data has negative total_spent.

## Step 3: Break it on purpose

To verify the test does what we think, let's make it fail.

Open `models/marts/dim_customers.sql` and change:
```sql
coalesce(co.total_spent, 0) as total_spent
```
to:
```sql
coalesce(co.total_spent, -999) as total_spent
```

(Now customers with no orders get `total_spent = -999` instead of `0`.)

Rebuild and re-test:

```bash
dbt build
```

You'll see:

```
... dim_customers (table) ... OK
... assert_dim_customers_total_spent_non_negative ... FAIL N
```

`FAIL N` where N is the number of customers with no orders (got -999 instead of
0). Your custom test caught it.

To see the failing rows, inspect the compiled SQL:

```bash
cat target/compiled/my_first_project/tests/assert_dim_customers_total_spent_non_negative.sql
```

You'll see:
```sql
-- Assert: every customer's total_spent is >= 0
select
    customer_id,
    total_spent
from `your-project.dbt_tutorial.dim_customers`
where total_spent < 0
```

Run that in BQ; you'll see customers with `total_spent = -999`.

**Revert** the model: change `-999` back to `0`. Re-run `dbt build`. Test passes.

## When to use a custom test vs a built-in

| Use a built-in (in YAML) | Use a custom test (in `tests/`) |
|---|---|
| Standard checks — not_null, unique, foreign keys, enums | Custom business invariants |
| One assertion per column | An assertion spanning multiple columns or rows |
| You want test names auto-generated and consistent | You want a meaningful test name |
| Reusable across many models? Yes, built-ins are reused | One-off — this assertion is specific to one model |

Roughly: **start with built-ins, escalate to custom only when built-ins can't
express what you need.**

## Generic tests: custom logic, reusable

You can also write **generic tests** — custom test logic that's reusable across
columns and models, just like the four built-ins.

A generic test is a macro in `tests/generic/` (or `macros/`) that takes a
`model:` and `column_name:` argument. dbt builds the test query from your
template.

Example skeleton:

```sql
-- macros/test_not_negative.sql
{% test not_negative(model, column_name) %}
    select {{ column_name }}
    from {{ model }}
    where {{ column_name }} < 0
{% endtest %}
```

Then in any YAML:

```yaml
columns:
  - name: total_spent
    data_tests:
      - not_negative
```

This kind of "I want a reusable test for a pattern that comes up in many places"
is generic test territory.

For most projects: a few custom singular tests + many built-in tests is the right
mix. You'll only build generic tests when you have a clear reusable pattern.

## A real example you might see

spotify-pipeline doesn't have custom tests in its current state. But a typical
real project has things like:

```sql
-- tests/assert_fct_orders_revenue_matches_source.sql
-- Sanity: total revenue in our fact equals total amount in the source.
-- Catches anything we accidentally filter or duplicate.
with fct as (
    select sum(amount) as fct_revenue from {{ ref('fct_orders') }}
),
src as (
    select sum(amount) as src_revenue from {{ source('raw', 'orders') }}
    where status = 'completed'
)
select fct_revenue, src_revenue
from fct, src
where fct_revenue != src_revenue
```

This is the kind of "sanity check across layers" that's impossible with
built-ins. The test passes iff the mart's revenue equals what we'd expect from
the source.

## What did you just do?

- Wrote your first custom (singular) test as a `.sql` file in `tests/`.
- Ran it and saw it pass.
- Broke a model on purpose and watched the test catch the bug.
- Reverted.
- Learned the conceptual difference between custom singular tests and generic
  test macros.

## Common confusion

> "Why is the file called `tests/assert_X.sql` and not `models/.../test_X.sql`?"

Convention. Files in `tests/` are tests; files in `models/` are models. dbt
treats them differently — a model becomes a table/view in your warehouse; a test
becomes a query that's only run when you `dbt test`.

> "If my test query returns 'extra' columns (e.g., I selected `customer_id,
> total_spent` not just one column), is that OK?"

Yes. dbt only cares whether the query returns any rows at all. The columns don't
matter — they're useful for human inspection when the test fails, that's all.

> "What's the difference between a singular test and a generic test?"

- **Singular**: a SQL file in `tests/`. One specific assertion, one model. Not
  reusable.
- **Generic**: a macro that takes a model+column+args. Reusable as `- my_test:` in
  any model's YAML. The four built-ins (`not_null`, etc.) are generic tests dbt
  ships with.

You start with singular. You upgrade to generic when you've written the same
pattern three times.

> "Can I have a custom test with `--store-failures`?"

Yes — `dbt test --select assert_X --store-failures` writes the failing rows to
`dbt_test__audit.assert_X`. Useful when CI runs and you want to debug after.

---

You can now write any test you can think of. Time to actually run all the tests.

← Previous&nbsp; [`18-column-tests.md`](18-column-tests.md) &nbsp;·&nbsp; Next →&nbsp; [`19-running-tests.md`](19-running-tests.md)
