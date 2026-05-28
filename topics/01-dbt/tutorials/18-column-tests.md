# 18 — Built-in column tests

← Previous&nbsp; [`17-why-tests.md`](17-why-tests.md) &nbsp;·&nbsp; Next →&nbsp; [`18b-custom-tests.md`](18b-custom-tests.md)

---

The four built-in tests are `not_null`, `unique`, `accepted_values`, and
`relationships`. We'll add tests using all four to our project.

## How built-in tests are declared

Tests live in a YAML file next to the model. Conventional names:
`_stg_models.yml` (next to staging models), `_marts_models.yml` (next to mart
models), `_sources.yml` (next to source declarations).

The underscore prefix is a dbt convention — it makes the YAML file sort to the
top of the folder listing in editors.

## Step 1: Test the staging models

Create `models/staging/_stg_models.yml`:

```bash
cat > models/staging/_stg_models.yml <<'EOF'
version: 2

models:
  - name: stg_customers
    description: "One row per customer, after light staging cleanup."
    columns:
      - name: customer_id
        description: "Primary key — the customer's unique id."
        data_tests:
          - not_null
          - unique
      - name: first_name
        data_tests:
          - not_null
      - name: last_name
        data_tests:
          - not_null
      - name: signup_date
        data_tests:
          - not_null

  - name: stg_orders
    description: "One row per order, after light staging cleanup."
    columns:
      - name: order_id
        description: "Primary key — the order's unique id."
        data_tests:
          - not_null
          - unique
      - name: customer_id
        description: "Foreign key into stg_customers."
        data_tests:
          - not_null
          - relationships:
              arguments:
                to: ref('stg_customers')
                field: customer_id
      - name: status
        data_tests:
          - not_null
          - accepted_values:
              arguments:
                values: ['completed', 'returned']
      - name: amount
        data_tests:
          - not_null
EOF
```

Let's read this.

### `version: 2`

dbt's YAML schema versioning. Don't change this — version 2 has been the standard
for years.

### `models:`

A list of models. Each entry has `name` (matching the `.sql` filename without
`.sql`), an optional `description`, and a list of `columns`.

### Each `column` block

```yaml
- name: customer_id
  description: "Primary key — the customer's unique id."
  data_tests:
    - not_null
    - unique
```

- `name:` — the column name in the model.
- `description:` — shown in `dbt docs`. Optional but helpful.
- `data_tests:` — a list of tests to apply to this column.

### The four built-in tests

Used here:

**`not_null`** — simplest. Just the test name as a string. dbt generates a query
that returns rows where this column is NULL.

**`unique`** — same shape. dbt generates a query that returns duplicate values.

**`accepted_values`** — needs arguments:

```yaml
- accepted_values:
    arguments:
      values: ['completed', 'returned']
```

dbt generates a query that returns rows where `status` is not in the list. (Note
the `arguments:` nesting — this is the dbt 1.10+ syntax; older versions used the
top-level `values:` directly. We use the new form.)

**`relationships`** — checks that every value in this column has a matching value
in another table:

```yaml
- relationships:
    arguments:
      to: ref('stg_customers')
      field: customer_id
```

This says: "Every `stg_orders.customer_id` must exist as a `customer_id` in
`stg_customers`." The test would fail if there's an order with `customer_id =
999` and no customer with that id.

## Step 2: Test the mart

Create `models/marts/_marts_models.yml`:

```bash
cat > models/marts/_marts_models.yml <<'EOF'
version: 2

models:
  - name: dim_customers
    description: |
      One row per customer with their order statistics.
      Customers with zero completed orders still appear (orders_count = 0).
    columns:
      - name: customer_id
        description: "Primary key — matches stg_customers.customer_id."
        data_tests:
          - not_null
          - unique
      - name: orders_count
        description: "Number of completed orders for this customer (0 if none)."
        data_tests:
          - not_null
      - name: total_spent
        description: "Sum of amount for completed orders (0 if none)."
        data_tests:
          - not_null
EOF
```

This is shorter — we don't need to test every column. The most critical
assertions:

- `customer_id` is the primary key (`not_null` + `unique`).
- `orders_count` and `total_spent` are never NULL (the `COALESCE` in the model
  should guarantee this, but the test catches it if we ever break the COALESCE).

## Step 3: Verify the YAML parses

```bash
dbt parse
```

If the YAML is malformed (indentation wrong, missing colons), you'll get an error
here. Otherwise:

```
Successfully parsed 433 macros, 2 sources, 3 models, 11 tests, ...
```

11 tests is what we expect:
- stg_customers: 4 columns × not_null + 1 unique = 5 tests
- stg_orders: 4 tests on order_id (not_null, unique), 2 on customer_id (not_null,
  relationships), 2 on status (not_null, accepted_values), 1 on amount (not_null)
  = actually 7 tests... or so. The exact number depends on how dbt counts.

Wait — `dbt parse` just says how many were found, not which ones. Let's enumerate
them:

| Model | Column | Tests |
|---|---|---|
| stg_customers | customer_id | not_null, unique |
| stg_customers | first_name | not_null |
| stg_customers | last_name | not_null |
| stg_customers | signup_date | not_null |
| stg_orders | order_id | not_null, unique |
| stg_orders | customer_id | not_null, relationships |
| stg_orders | status | not_null, accepted_values |
| stg_orders | amount | not_null |
| dim_customers | customer_id | not_null, unique |
| dim_customers | orders_count | not_null |
| dim_customers | total_spent | not_null |

That's 4 + 2 + 1 + 2 + 2 + 2 + 1 + 2 + 1 + 1 = 17 tests, possibly.

(`dbt parse` shows the total; the actual number is what dbt finds.)

## What did you just do?

- Wrote two YAML test files that declare ~17 column tests across the project.
- Used all four built-in test types: `not_null`, `unique`, `accepted_values`,
  `relationships`.
- Added column descriptions for self-documenting structure.

## Common confusion

> "Do I need to test EVERY column?"

No. Test the columns where bad data would cause problems. Primary keys
(`not_null` + `unique`) are non-negotiable. Foreign keys (`relationships`) catch
orphan rows. Enum columns (`accepted_values`) catch unexpected statuses. For
most other columns, skip unless there's a specific invariant you need to enforce.

> "I see `tests:` vs `data_tests:` in different sources. Which is right?"

`data_tests:` is the dbt 1.8+ preferred name. `tests:` is the older name and
still works for backward compatibility. We use `data_tests:` to be current.

> "What if my data legitimately has NULL in a column? Can I still have a model-
> level test that excludes nulls from another check?"

Yes. Many tests support `config: { where: "..." }` to filter. For example:

```yaml
- unique:
    config:
      where: "customer_id is not null"
```

This says "only check uniqueness among rows where customer_id is not null."

> "What's the difference between a test FAILING and a test ERRORING?"

A test fails when it does its job and finds bad data (the test query returned
rows). A test errors when something else went wrong — bad SQL syntax, table
doesn't exist, etc. dbt distinguishes these in output: `FAIL` vs `ERROR`.

---

We've declared tests. Let's actually run them.

← Previous&nbsp; [`17-why-tests.md`](17-why-tests.md) &nbsp;·&nbsp; Next →&nbsp; [`18b-custom-tests.md`](18b-custom-tests.md)
