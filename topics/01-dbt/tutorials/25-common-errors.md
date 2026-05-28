# 25 — Common errors (reference)

← Previous&nbsp; [`24-dbt-docs.md`](24-dbt-docs.md) &nbsp;·&nbsp; Next →&nbsp; [`26-whats-next.md`](26-whats-next.md)

---

This is a reference file. Don't read it top-to-bottom. Skim the headings, come
back when stuck.

## Setup errors

### `Could not find profile named 'my_first_project'`

dbt looked in `~/.dbt/profiles.yml` for a top-level key matching the `profile:`
field in your `dbt_project.yml`. The names don't match.

Check:
```bash
grep '^profile:' dbt_project.yml
head -1 ~/.dbt/profiles.yml
```

The strings (after stripping whitespace) must match.

### `BigQuery client configuration error: Cannot find credentials`

ADC isn't set up. Run:
```bash
gcloud auth application-default login
```

### `403 ... User does not have permission ... project: foo`

Either:
- The project ID in `profiles.yml` is wrong (you typed it wrong, or you're
  pointing at someone else's project).
- ADC is authenticated as a different Google account than the one with access
  to that project.

Check the current ADC user:
```bash
gcloud auth list
```

Re-authenticate if needed:
```bash
gcloud auth application-default login
```

### `Quota project ... was not found`

ADC's quota project points at a deleted project. Re-set it:
```bash
gcloud auth application-default set-quota-project YOUR_PROJECT_ID
```

### `dbt: command not found`

Your venv isn't active. `cd` into your project dir and:
```bash
source .venv/bin/activate
```

### `ModuleNotFoundError: No module named 'dbt'`

You ran `python` or `pip` from outside the venv. Always:
```bash
source .venv/bin/activate
dbt --version
```

## Model errors

### `Compilation Error in model X: 'foo' is undefined`

dbt's Jinja couldn't find a reference. Common causes:

- Typo in `{{ ref('foo') }}` — the model named `foo` doesn't exist.
- Forgot quotes: `{{ ref(foo) }}` — Python error.
- Macros not loaded (rare; `dbt clean` then re-run).

Check what dbt thinks exists:
```bash
dbt list --resource-type model
```

### `Compilation Error: dbt found a cycle: A → B → A`

You created a cyclic dependency. Two models reference each other. Fix the
dependency graph — one of the refs has to go.

### `Database Error in model X: Not found: Dataset ... was not found in location US`

The BigQuery dataset doesn't exist yet AND your `location:` doesn't match BQ's
default.

Make sure `location: US` is set in `~/.dbt/profiles.yml`. dbt will create the
dataset on first run if it doesn't exist.

### `Database Error in model X: Not found: Table ... was not found in location US`

A `{{ ref('foo') }}` resolved to a table that doesn't exist in BigQuery yet. Most
common cause: you ran the dependent model without first running the upstream
model (or seed). Fix: `dbt build` (which builds in dependency order) or
`dbt run --select +my_model` (model and its upstream).

### `Compilation Error in model X: Depends on a node named raw_customers which was not found`

You used `{{ ref('raw_customers') }}` but dbt doesn't know about `raw_customers`.
Either:
- The seed file isn't in `seeds/`.
- You renamed the seed file but not the ref call.
- You haven't run `dbt seed` yet (dbt parses seeds the same as models; just needs
  the file to exist, not yet loaded).

## YAML errors

### `Parsing Error: ... while parsing a block mapping`

Your YAML is malformed. Common causes:

- **Indentation**: YAML uses spaces, not tabs. 2-space indents recommended.
- **Missing colons**: every key needs `:`.
- **Misaligned items**: a list item under the wrong key.

Open the file at the line dbt mentions. Use a YAML validator (or your editor's
syntax highlighting) to find the issue.

### `accepted_values: ... Validation Error`

Old `values:` syntax instead of the new `arguments:` nesting. Wrong:

```yaml
- accepted_values:
    values: ['a', 'b']
```

Right:

```yaml
- accepted_values:
    arguments:
      values: ['a', 'b']
```

## Test errors

### `FAIL N`

The test query returned N rows. Real bad data — investigate.

To see the bad rows:
1. Open `target/compiled/.../tests/that_test_name.sql`.
2. Run that SQL in BigQuery.
3. Look at the rows that come back. Those are the violations.

Or use `--store-failures`:
```bash
dbt test --store-failures
```

dbt creates `dbt_test__audit.<test_name>` tables with the failing rows. Query
those after a fail.

### `ERROR ... <something>`

The test couldn't even run (vs. ran and found bad data). Reasons:

- The test references a table that doesn't exist.
- SQL syntax error in a custom test.
- Permission denied to query the table.

Read the actual error message — it's usually specific.

## Permission / IAM errors

### `403 ... User does not have bigquery.jobs.create`

The Google account ADC is using doesn't have permission to run BQ jobs in the
project. For sandbox projects, your user should have full access. Check `gcloud
auth list` and `gcloud auth application-default print-access-token`.

### `403 ... User does not have bigquery.datasets.create`

Same idea — your user can't create datasets. For sandbox, this should not happen.
For real projects with role-based access: ask the admin to grant
`roles/bigquery.user` or `roles/bigquery.dataEditor`.

## Seed errors

### `Database Error in seed raw_customers: invalid value for ... at column N`

dbt's auto-type-detection got something wrong. The most common cause: a column
that's mostly numbers but has one row with text (or vice versa). dbt assumed
numeric, BigQuery rejects the text row.

Fix by declaring the column type in `_seeds.yml`:

```yaml
seeds:
  - name: raw_customers
    config:
      column_types:
        weird_column: STRING
```

### `seed file dbt_tutorial.raw_X failed`

Usually a CSV parsing issue. Common causes:

- Commas inside a field without quotes: `1,Phillips, Jr.,...`
- Mismatched column count between header and rows.

Fix the CSV. Use double quotes around fields with commas: `1,"Phillips, Jr.",...`.

## Network / runtime errors

### `Database Error: 502 Bad Gateway` or `503 Service Unavailable`

BigQuery had a temporary hiccup. Retry the command. If it persists, check
https://status.cloud.google.com.

### `Database Error: timeout`

Your query took longer than `timeout_seconds` in `profiles.yml` (default 300).
For complex queries: bump the timeout. For "this should be fast but isn't":
investigate why.

### `MaxRetryError ... new connection: refused`

Network or DNS issue on your laptop. Often a VPN flaking out. Restart your
network connection and retry.

## When in doubt

1. **Read the actual error message.** dbt's errors are pretty descriptive.
2. **Look at compiled SQL** in `target/compiled/.../` to see what dbt actually
   sent to BigQuery.
3. **Run that SQL directly in the BigQuery console** to see if it's a dbt
   issue or a BigQuery issue.
4. **Check `dbt debug`** — if your setup is broken, this catches it.

If none of those help: post on https://discourse.getdbt.com/. The dbt community
is huge and responsive.

---

Last file — what to do after this tutorial.

← Previous&nbsp; [`24-dbt-docs.md`](24-dbt-docs.md) &nbsp;·&nbsp; Next →&nbsp; [`26-whats-next.md`](26-whats-next.md)
