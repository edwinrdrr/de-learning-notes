# 10 — `dbt run` — what it actually does

← Previous&nbsp; [`09-the-example-models.md`](09-the-example-models.md) &nbsp;·&nbsp; Next →&nbsp; [`11-seeds.md`](11-seeds.md)

---

We've understood the examples and deleted them. Before we write our own models,
let's run dbt one more time with the empty project to see exactly what `dbt run`
does — and to inspect what dbt writes to disk along the way.

## What `dbt run` does, conceptually

`dbt run` is the workhorse. In order, it:

1. **Parses** every file in your project. Reads `dbt_project.yml`, every `.sql`,
   every `.yml`. Builds an internal representation of the whole project — what
   models exist, what they depend on.
2. **Compiles** every model. Renders the Jinja (`{{ ref(...) }}`, etc.) into plain
   SQL with full table references.
3. **Plans** the build order from the dependency graph (the DAG).
4. **Connects** to BigQuery.
5. **Executes** the models in order. For each one, sends the compiled SQL as a
   `CREATE TABLE AS SELECT ...` or `CREATE VIEW AS ...` to BigQuery.

Note: **`dbt run` does NOT run tests.** That's a separate command (`dbt test`).
We'll see the combined `dbt build` command in file 21.

## Step 1: Run it with an empty project

From inside `my_first_project/` (venv active):

```bash
dbt run
```

Expected output (paraphrased):

```
Running with dbt=1.11.x
Found 0 models, 0 measures, 0 tests, 0 snapshots, ...
Concurrency: 4 threads (target='dev')

Nothing to do. Try checking your model selection or model name pattern.
```

dbt says "Found 0 models" because we deleted the example folder and have nothing
else. It still connected, parsed, and acknowledged the project — but there was no
work to do.

This is fine. It confirms dbt works end-to-end.

## Step 2: Look at the `target/` folder

dbt always writes its scratch work to a folder called `target/` inside the project.
Let's see what showed up:

```bash
ls target/
```

You'll see:

```
graph.gpickle
manifest.json
partial_parse.msgpack
run_results.json
semantic_manifest.json
```

What these are:

- **`manifest.json`** — dbt's internal representation of your project: every
  model, every test, every source, every macro, the full DAG. Other dbt tools
  (`dbt docs`, Slim CI, third-party integrations) read this. We'll use it later for
  Slim CI.
- **`partial_parse.msgpack`** — a cache that makes the next `dbt parse` faster.
- **`run_results.json`** — what happened on the most recent run (which models
  built, how long they took, whether they succeeded).
- **`graph.gpickle`** — a serialized form of the DAG, used internally.
- **`semantic_manifest.json`** — newer (introduces the semantic layer / metrics).
  We won't touch this.

## Step 3: Other folders that show up after a real run

After you actually have models that build, two more folders appear under
`target/`:

- **`target/compiled/`** — your `.sql` files after Jinja is rendered. Plain SQL,
  ready to send to BigQuery. **This is gold for debugging.** When a query fails
  and the error makes no sense, open the file at
  `target/compiled/my_first_project/models/.../my_model.sql` and look at the
  actual SQL dbt ran.
- **`target/run/`** — the wrapped versions (with `CREATE TABLE AS` etc. around the
  compiled SQL). What was actually executed.

You can't see these yet because we haven't successfully built anything.

## Step 4: Useful flags on `dbt run`

A few flags you'll reach for:

```bash
# Only run a specific model (and its dependencies)
dbt run --select my_model

# Only run everything in a folder
dbt run --select staging

# A model AND everything downstream of it
dbt run --select my_model+

# A model AND everything upstream
dbt run --select +my_model

# Exclude something
dbt run --exclude marts

# Use a different target (from profiles.yml outputs)
dbt run --target prod

# Don't actually run, just compile (writes to target/compiled/)
dbt compile
```

The `--select` selectors are powerful — `dbt run --select state:modified.body+` is
how Slim CI picks only the changed models. We'll see selectors in detail later.

## Step 5: `dbt run` vs `dbt compile`

If you ever want to see what SQL dbt is going to send, without actually sending
it, use `dbt compile`:

```bash
dbt compile
```

This does steps 1-3 above (parse, compile, plan) but skips connecting and executing.
You can then read the rendered SQL files in `target/compiled/` to debug or to copy
into the BigQuery console to manually test.

Real workflow:
1. Write or edit a model.
2. `dbt compile` to render it.
3. Read the rendered SQL — make sure it's what you intended.
4. `dbt run` to actually execute.

## Step 6: A note about state

`dbt run` *replaces* the existing model in BigQuery each time:

- For `table` materializations: drops the existing table, creates fresh.
- For `view` materializations: drops the existing view, creates fresh.
- For `incremental` materializations: appends/merges new rows.

This means **`dbt run` is idempotent** for non-incremental models. Run it 5 times,
end state is identical to running it once. Useful when you're iterating: tweak
SQL, re-run, see new result.

## What did you just do?

- Ran `dbt run` against an empty project and saw what it does without any work to
  do.
- Met the `target/` folder and what dbt writes there.
- Learned the difference between `dbt run` and `dbt compile`.
- Saw the most common `--select` patterns.

## Common confusion

> "I see `target/` in my project. Should I commit it to git?"

No. `target/` is build output. It gets regenerated every run. The standard
`.gitignore` for a dbt project includes:

```
target/
dbt_packages/
logs/
```

> "Why did `dbt run` print 'Found 0 models, 0 measures, 0 tests' when there are
> still empty folders for snapshots, macros, etc.?"

Because those folders are empty — no actual files in them. dbt counts files, not
folders. Empty `tests/`, `macros/`, etc. produce 0 of each. That's fine.

> "If I make a typo in a model file, does dbt tell me before executing?"

Yes. `dbt run` parses and compiles before any model executes. If a model has a
syntax error in its `{{ }}` Jinja or a typo in `ref('typo_here')`, dbt errors out
at compile time, before sending anything to BigQuery. No half-state.

> "Can I run dbt run without an internet connection?"

No. dbt needs to talk to BigQuery to do anything useful. `dbt compile` does NOT
need internet (it just renders local files), so you can compile offline to check
your work.

---

We've seen `dbt run` do nothing. Now let's give it something to do.

← Previous&nbsp; [`09-the-example-models.md`](09-the-example-models.md) &nbsp;·&nbsp; Next →&nbsp; [`11-seeds.md`](11-seeds.md)
