# 17 — Why data tests exist

← Previous&nbsp; [`16-the-dag.md`](16-the-dag.md) &nbsp;·&nbsp; Next →&nbsp; [`18-column-tests.md`](18-column-tests.md)

---

We have a working pipeline. It produces an answer. But — is the answer *correct*?

If you stop here, you're trusting that:
- The CSVs you uploaded match reality.
- The seeds loaded without dropping or corrupting rows.
- Your staging models didn't accidentally filter something.
- The join in `dim_customers` didn't fan out (each customer row should appear
  exactly once).
- The `COALESCE` defaults are sensible.

You have no way to verify any of this **automatically**. If a bug crept in, your
dashboard would silently show wrong numbers. Stakeholders would make decisions
on those numbers. By the time someone noticed, weeks would have passed.

This is the silent-bad-data problem. dbt tests are how you guard against it.

## What is a dbt test?

A **dbt test** is a SQL query that *should return zero rows* if your data is
healthy. If it returns any rows, the test failed — those rows are the violations.

That's the entire mechanism. Tests are queries that "no rows means OK."

Examples:

| Test | What it queries | Failure means |
|---|---|---|
| `not_null` on `id` | `SELECT id FROM table WHERE id IS NULL` | Some rows have null id |
| `unique` on `id` | `SELECT id FROM table GROUP BY id HAVING COUNT(*) > 1` | Duplicate ids exist |
| `accepted_values` on status | `SELECT status FROM table WHERE status NOT IN ('a','b','c')` | Unexpected values |
| `relationships` (customer_id → stg_customers.customer_id) | `SELECT o.customer_id FROM orders o LEFT JOIN customers c USING(customer_id) WHERE c.customer_id IS NULL` | Orders with no matching customer |

dbt has four "built-in" tests (`not_null`, `unique`, `accepted_values`,
`relationships`) plus you can write your own custom tests.

## Two flavors of tests

### Built-in column tests

Declared in a `.yml` file next to the model:

```yaml
models:
  - name: stg_customers
    columns:
      - name: customer_id
        data_tests:
          - not_null
          - unique
```

No SQL writing required. dbt generates the test query for you. Use these for the
70% of cases that fit a built-in pattern.

### Custom tests

You write a `.sql` file in `tests/` (project-level singular tests) or in a
`tests/generic/` macro (reusable across models). Useful for assertions the built-ins
can't express:

```sql
-- tests/dim_customers_total_spent_positive.sql
select customer_id, total_spent
from {{ ref('dim_customers') }}
where total_spent < 0
```

The test "passes" if it returns 0 rows (no customer has negative total spent).

We won't write custom tests in this tutorial — built-ins cover the common cases.
But know they exist.

## Where to put tests in the data flow

A common pattern:

- **Source tests** (in a `_sources.yml`): test the raw data before any
  transformation. `not_null` on key columns of raw tables catches ingestion bugs.
- **Staging model tests**: test the cleaned data. After staging, the data should
  have proper types, no obvious garbage.
- **Mart tests**: test the analytics output. Joins didn't fan out. Statuses are
  in expected values. Foreign keys resolve.

You don't need tests at every layer for every column. Focus on:

- **Primary keys** (must be `not_null` and `unique`).
- **Foreign keys** (`relationships` test — there should be no orphans).
- **Enum-like columns** (`accepted_values` — status should be one of a known set).
- **Critical business invariants** (custom tests — "no negative revenue", "total
  shipments = sum of per-day shipments", etc.).

## When tests run

Tests don't run automatically as part of `dbt run`. They have their own command:

```bash
dbt test          # run all tests
dbt build         # run + test in one command, in dependency order
```

For day-to-day use, you'll run `dbt build` — it does run and test together. We'll
see this in file 21.

## What makes a test "good"?

A test is only useful if it actually catches problems. Two checks:

1. **It should fail when data is bad.** If you can break the data on purpose and
   the test stays green, it's broken or pointing at the wrong thing.
2. **It shouldn't be flaky.** A test that fails intermittently for "real" data is
   worse than no test — it teaches you to ignore failures, masking the real one.

The first time you add a test to a model, **try to break it on purpose** (file
20). If it doesn't fail when you'd expect it to, you've found a bug in the test.

## Test as documentation

Tests are also documentation. When someone reads `_stg_models.yml` and sees:

```yaml
- name: customer_id
  data_tests: [not_null, unique]
```

They learn, instantly, that `customer_id` is the primary key. No need to dig
through SQL to figure it out. The test IS the assertion.

This is part of why people put tests in `.yml` instead of separate `.sql`
files — the structure of the project documents itself.

## What did you just learn?

- A dbt test is a SQL query that returns zero rows if data is healthy.
- Built-in tests (`not_null`, `unique`, `accepted_values`, `relationships`) cover
  the common cases without writing SQL.
- Custom tests exist for everything else.
- Tests guard against silent-bad-data — you find problems automatically instead of
  watching dashboards lie.
- `dbt test` runs them; `dbt build` runs models then tests in dependency order.
- A good test fails when data is bad; verify by breaking the data on purpose.

## Common confusion

> "How is this different from unit tests in software?"

A unit test in software tests **code**: given inputs, does the function produce
the expected output? A data test tests **data**: do the rows in this table
satisfy this invariant? They feel similar but operate on different things.

dbt does also support unit tests on models (a newer feature — `dbt test --resource-type
unit_test`). For this tutorial we're using only data tests.

> "How many tests is enough?"

There's no number. The right answer is "every assumption you've made about the
data, written as a test." But that's a lot, and most projects don't have time.
Practical default: test primary keys (not_null + unique) on every model, plus
foreign-key `relationships` tests across joins, plus `accepted_values` on any
enum-like columns.

> "Tests slow down the build, right?"

Yes, but not much. Each test is a SQL query — quick. Even for big projects with
hundreds of tests, the overhead is usually under 10% of total build time. The
trade is worth it.

---

Enough theory. Let's add some tests.

← Previous&nbsp; [`16-the-dag.md`](16-the-dag.md) &nbsp;·&nbsp; Next →&nbsp; [`18-column-tests.md`](18-column-tests.md)
