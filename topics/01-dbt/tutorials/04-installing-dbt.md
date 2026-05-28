# 04 — Install dbt

← Previous&nbsp; [`03-python-venv.md`](03-python-venv.md) &nbsp;·&nbsp; Next →&nbsp; [`05-dbt-init.md`](05-dbt-init.md)

---

Your virtualenv is active. Time to put dbt inside it.

## Two pieces of dbt

dbt is split into two packages:

- **`dbt-core`** — the engine. Parses your project, compiles Jinja, builds the DAG,
  runs tests. It's database-agnostic.
- **`dbt-<warehouse>`** — an **adapter** for a specific database. Knows how to
  translate dbt's intent into SQL that BigQuery (or Snowflake, or Postgres)
  understands.

Adapters available: `dbt-bigquery`, `dbt-snowflake`, `dbt-redshift`, `dbt-postgres`,
`dbt-duckdb`, and more. You pick one.

You don't have to install `dbt-core` separately — when you install any adapter,
`dbt-core` is automatically pulled in as a dependency.

## Install `dbt-bigquery`

Make sure your venv is still active (you should see `(.venv)` in your prompt). If
not:

```bash
cd ~/learning/dbt-tutorial
source .venv/bin/activate
```

Now install:

```bash
pip install dbt-bigquery
```

This takes ~30 seconds. You'll see pip download a bunch of packages. The output
ends with something like:

```
Successfully installed Jinja2-3.x.x agate-1.x.x ... dbt-bigquery-1.9.x dbt-core-1.11.x google-cloud-bigquery-3.x.x ...
```

What got installed:
- `dbt-bigquery` — the BigQuery adapter (the one we asked for).
- `dbt-core` — automatically pulled in.
- `Jinja2` — the templating engine. dbt uses Jinja for `{{ ref(...) }}` and similar.
  We'll see this in file 12.
- `google-cloud-bigquery` — Google's official Python library for BQ. dbt's BQ
  adapter uses this under the hood.
- A pile of supporting libraries.

All of them go into `.venv/lib/python3.x/site-packages/`. They're isolated to this
venv.

## Verify

```bash
dbt --version
```

Expected output:

```
Core:
  - installed: 1.11.x
  - latest:    1.11.x - Up to date!

Plugins:
  - bigquery: 1.9.x - Up to date!
```

(Exact version numbers will vary depending on when you install. As long as you see
the two lines and no errors, you're good.)

## What `dbt --version` told you

- **`Core: 1.11.x`** — the dbt engine version.
- **`Plugins: bigquery: 1.9.x`** — the adapter version. dbt-bigquery and dbt-core are
  versioned separately but should usually be released in the same era.
- **`Up to date!`** — dbt checked PyPI for newer versions and didn't find any. If you
  install months from now and a newer version is out, it'll say so. That's fine for
  this tutorial; we want a stable version.

## Step 4: See where dbt was installed

```bash
which dbt
```

Expected: a path inside your venv, like
`/home/edwin/learning/dbt-tutorial/.venv/bin/dbt`.

If `which dbt` returns nothing or points outside the venv, something went wrong.
The most common cause: you ran `pip install` from outside the venv. Solution:
re-activate, re-install.

## What did pip actually do?

Walking the steps:

1. **Resolved dependencies**: `dbt-bigquery 1.9.x` needs `dbt-core 1.11.x`,
   `google-cloud-bigquery 3.x`, and so on. pip figured out a set of versions that
   work together.
2. **Downloaded** each one from PyPI (Python's package index, `pypi.org`).
3. **Installed** each into `.venv/lib/python3.x/site-packages/`.
4. **Created CLI entry points**: when you install `dbt-core`, pip adds a `dbt`
   script to `.venv/bin/`. That's what `dbt --version` invokes.

The "CLI entry point" detail is why we have a `dbt` command at all — it's a tiny
Python script that imports `dbt.cli.main:cli` and runs it.

## What did you just do?

- Installed `dbt-bigquery` (and its dependency `dbt-core`) into your venv.
- Verified that the `dbt` command works and points at the venv.
- Got familiar with what pip did under the hood.

## Common confusion

> "`pip install dbt-bigquery` is slow / it's downloading hundreds of MB."

Normal. dbt has a few dozen transitive dependencies. First install pulls them all
fresh. Future installs of dbt-related stuff into other venvs will be faster
because pip caches packages locally.

> "I get an error about `pyodbc` or `oscrypto`."

These are dependencies of other dbt adapters (Snowflake/Redshift). They sometimes
get pulled in by mistake. For pure dbt-bigquery installation, you shouldn't see
them. If you do, double-check you typed `dbt-bigquery` (with the dash).

> "Can I install multiple adapters at once?"

Yes — `pip install dbt-bigquery dbt-postgres` works. dbt-core gets installed once;
both adapters share it. The `dbt` command then knows about both warehouse types.

> "What's the difference between `dbt` and `dbt-core` as packages?"

There used to be a `dbt` package on PyPI that was just an alias bundling
everything. It's deprecated. Now you install adapters directly (`dbt-bigquery`,
etc.), and they bring in `dbt-core`. Don't `pip install dbt` by itself — install
the adapter.

> "Should I freeze the version, like `dbt-bigquery==1.9.0`?"

For real projects, yes — pin the version so reproducible builds are actually
reproducible. For learning, the latest works fine.

---

dbt is installed. Time to make a project.

← Previous&nbsp; [`03-python-venv.md`](03-python-venv.md) &nbsp;·&nbsp; Next →&nbsp; [`05-dbt-init.md`](05-dbt-init.md)
