# 23b — Packages and macros

← Previous&nbsp; [`23-configuring-mat.md`](23-configuring-mat.md) &nbsp;·&nbsp; Next →&nbsp; [`23c-multi-target.md`](23c-multi-target.md)

---

You've been using dbt's built-in features. The dbt ecosystem also has
**packages** — pre-built libraries of macros, tests, and sometimes models you
can install and use, like npm or pip for dbt.

This file teaches you how to install one (`dbt_utils`, the most popular) and use
it. As a bonus, it introduces **macros** — your own custom Jinja functions.

## What is a package?

A package is a folder of dbt files (macros, models, tests) you can install into
your project via `packages.yml`. Once installed, your models can use the
package's contents as if they were your own.

The biggest one: **`dbt_utils`**. It has dozens of helpful macros like
`generate_surrogate_key()`, `date_spine()`, `pivot()`, and more.

Other useful packages:
- `dbt_expectations` — extra test types (Great-Expectations-style).
- `audit_helper` — for diffing two models to verify a refactor didn't change output.
- `dbt_external_tables` — for declaring sources that are external (e.g., S3 files).

For this tutorial we'll install `dbt_utils` and use one of its functions.

## Step 1: Declare the package

Create `packages.yml` at the **root** of your dbt project (same level as
`dbt_project.yml`):

```bash
cat > packages.yml <<'EOF'
packages:
  - package: dbt-labs/dbt_utils
    version: 1.3.0
EOF
```

That's the entire declaration. `package:` is the package's full name (org/name);
`version:` pins which version to install.

## Step 2: Install it

```bash
dbt deps
```

What dbt does:
1. Reads `packages.yml`.
2. Downloads the listed packages from the **dbt Hub** (https://hub.getdbt.com).
3. Unpacks them into `dbt_packages/` in your project.
4. Updates your project's manifest so the new macros are available.

Expected output:

```
Running with dbt=1.11.x
Installing dbt-labs/dbt_utils
  Installed from version 1.3.0
  Up to date!
```

Look at what got installed:

```bash
ls dbt_packages/
# dbt_utils
ls dbt_packages/dbt_utils/macros/ | head
# (dozens of .sql files)
```

That's all the macros `dbt_utils` provides.

## Step 3: Use a `dbt_utils` macro

`generate_surrogate_key()` is one of the most-used. It produces a hashed key from
a list of columns — useful when your "natural key" is multiple columns and you
want a single column to join on.

Let's use it in `dim_customers`. Edit `models/marts/dim_customers.sql`. Find the
final SELECT and add a surrogate key column:

```sql
select
    {{ dbt_utils.generate_surrogate_key(['c.customer_id', 'c.signup_date']) }}
        as customer_surrogate_key,
    c.customer_id,
    c.first_name,
    ...
```

(Add `customer_surrogate_key` as the first column. The rest of the SELECT
unchanged.)

Rebuild:

```bash
dbt build --select dim_customers
```

Query the result in BigQuery:

```sql
SELECT customer_surrogate_key, customer_id, first_name
FROM `your-project.dbt_tutorial.dim_customers`
LIMIT 3
```

You'll see something like:

```
| customer_surrogate_key                       | customer_id | first_name |
|----------------------------------------------|-------------|------------|
| a3f5b1c2d4e5f6789...                         | 1           | Michael    |
| b4f6c2d3e5f6a78...                           | 2           | Shawn      |
```

`customer_surrogate_key` is a deterministic hash of (`customer_id`,
`signup_date`). Same input → same hash. Different inputs → different hash.

## Step 4: Inspect what `generate_surrogate_key` compiled to

Look at the compiled model:

```bash
cat target/compiled/my_first_project/models/marts/dim_customers.sql
```

You'll see the macro got replaced with a SQL expression — something like:

```sql
to_hex(md5(cast(coalesce(cast(c.customer_id as STRING), '') ||
  '-' || coalesce(cast(c.signup_date as STRING), '') as STRING)))
```

That's what `generate_surrogate_key` actually does: cast each column to string,
concat them with a separator, MD5-hash, hex-encode. You could write that
yourself — but writing `dbt_utils.generate_surrogate_key([...])` is shorter and
the macro guarantees consistency across all your models.

## Step 5: Revert (optional)

If you want to clean up:

1. Remove the `customer_surrogate_key` column from `dim_customers.sql`.
2. Run `dbt build --select dim_customers`.

The package stays installed (you can leave it for future use, or `rm -rf
dbt_packages/ packages.yml` to fully remove).

## Macros: writing your own

`dbt_utils` is a collection of macros made by dbt Labs. You can write **your own
macros** too — Jinja functions that produce SQL.

Macros live in `macros/`. Example:

```sql
-- macros/cents_to_dollars.sql
{% macro cents_to_dollars(column_name) %}
    ({{ column_name }} / 100.0)
{% endmacro %}
```

Then in a model:

```sql
select
    order_id,
    {{ cents_to_dollars('amount_cents') }} as amount_dollars
from ...
```

Compiles to:
```sql
select
    order_id,
    (amount_cents / 100.0) as amount_dollars
from ...
```

**When to write a macro**: when you find yourself writing the same SQL fragment
in three or more models. Factor it out, name it descriptively, use it everywhere.

**When NOT to write a macro**: for one-off logic. The indirection isn't worth it
for code that's only used once.

A real-world rule: **every business-specific calculation should be a macro**, so
when the business changes the formula, you change ONE file.

## Other things you can do with packages

- **Hub.getdbt.com** lists hundreds of packages. Many are warehouse-specific
  (`dbt-labs/spark_utils`, `calogica/dbt_expectations`, etc.).
- **Pinning versions** matters — use `version: 1.3.0`, not `version: '>=1.0.0'`,
  for reproducible builds.
- **`dbt clean`** does NOT remove `dbt_packages/`. Use `rm -rf dbt_packages/` if
  you want to wipe and reinstall.

## What did you just do?

- Created `packages.yml` and declared `dbt_utils` as a dependency.
- Ran `dbt deps` to install it.
- Used `dbt_utils.generate_surrogate_key()` in a model.
- Inspected the compiled SQL to see what the macro produced.
- Learned how to write your own macros for reusable SQL fragments.

## Common confusion

> "Do I need to commit `dbt_packages/` to git?"

No. Standard `.gitignore`:
```
dbt_packages/
target/
logs/
```

Commit `packages.yml` (declares what to install); regenerate `dbt_packages/`
via `dbt deps` on each fresh clone.

> "I get `Compilation Error: 'dbt_utils' is undefined`."

You forgot to run `dbt deps`, or you installed the package but haven't re-parsed.
Run `dbt deps`; if that doesn't fix it, `dbt clean && dbt deps`.

> "What's the difference between a macro and a Python function?"

A macro produces a string of SQL. The string is substituted into your model's
SQL at compile time. Macros run on your laptop (in dbt-core), not in the
warehouse. The output is plain SQL.

A Python function runs at... wait, that's not a thing dbt does by default.
There ARE "Python models" in newer dbt, but they're warehouse-side (BigQuery
runs the Python via DataFrames). Different beast; rarely needed.

> "Should I write macros for everything?"

No. Macros add indirection — when you read a model and see
`{{ my_macro(...) }}`, you have to go look up the macro to know what's happening.
Reserve macros for genuine reuse. For one-off logic, plain SQL is clearer.

---

Packages let you reuse community code. Macros let you reuse your own. Now let's
add a prod target to the project so we can see env-aware switching.

← Previous&nbsp; [`23-configuring-mat.md`](23-configuring-mat.md) &nbsp;·&nbsp; Next →&nbsp; [`23c-multi-target.md`](23c-multi-target.md)
