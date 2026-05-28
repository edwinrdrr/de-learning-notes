# 06 — `dbt_project.yml`

← Previous&nbsp; [`05-dbt-init.md`](05-dbt-init.md) &nbsp;·&nbsp; Next →&nbsp; [`07-profiles-yml.md`](07-profiles-yml.md)

---

You just ran `dbt init` and `cd`'d into the new folder. One of the files in there
is `dbt_project.yml`. Let's understand what it's for before we touch anything else.

## What it is, in one sentence

**`dbt_project.yml` is the file that turns a folder of SQL files into "a dbt
project."** Without it, dbt has no idea your folder exists. With it, dbt knows the
project's name, where to look for models, what to call the connection, and what
defaults to apply.

## Look at it

In your terminal, from inside the project folder:
```bash
cat dbt_project.yml
```

You'll see something like this (parts may differ slightly between dbt versions):

```yaml
name: 'my_first_project'
version: '1.0.0'
config-version: 2

profile: 'my_first_project'

model-paths: ["models"]
analysis-paths: ["analyses"]
test-paths: ["tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]

target-path: "target"
clean-targets:
  - "target"
  - "dbt_packages"

models:
  my_first_project:
    example:
      +materialized: view
```

That's a lot. Let's read it line by line.

## Line by line

### `name: 'my_first_project'`

The internal name of your project. You'll see this in error messages and in dbt
logs. It must be **lowercase, no spaces, only letters/numbers/underscores**.

This is *separate* from the folder name. You could rename the folder; the project
name stays whatever's in here.

### `version: '1.0.0'`

A version string for your dbt PROJECT (not for dbt-the-tool). Cosmetic — dbt
doesn't care what's here. Useful for humans reading the project history.

### `config-version: 2`

Tells dbt which version of the `dbt_project.yml` schema you're using. Version 2 has
been the standard since dbt 0.17 (a few years ago). You'll almost never need to
change this.

### `profile: 'my_first_project'`

**This is the bridge to `profiles.yml`.** When dbt needs to connect to your
warehouse, it does this:

1. Looks at the `profile:` field here. Gets the string `my_first_project`.
2. Opens `~/.dbt/profiles.yml`.
3. Looks inside for a top-level key with that same name (`my_first_project:`).
4. Reads the connection details from there.

So **`dbt_project.yml`'s `profile:` value must match a key in `profiles.yml`**. If
they don't match, dbt errors with `Could not find profile named '...'`.

We'll look at `profiles.yml` in the next file.

### `*-paths` fields

```yaml
model-paths: ["models"]
analysis-paths: ["analyses"]
test-paths: ["tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]
```

These tell dbt **where to look** for different kinds of files. The defaults match the
folder names that `dbt init` created. Most projects never change these.

What each is for (we'll touch most of them later):

- `model-paths` — your `.sql` model files. **The main event.**
- `analysis-paths` — one-off SQL queries that dbt compiles but doesn't run. Useful
  for "I want dbt to template this query so I can run it in BigQuery's UI."
- `test-paths` — custom SQL tests (vs the built-in column tests in `.yml`).
- `seed-paths` — CSV files dbt should load as tables (raw reference data).
- `macro-paths` — Jinja macros (reusable bits of SQL template logic).
- `snapshot-paths` — slowly-changing-dimension (SCD-2) snapshot definitions.

### `target-path` and `clean-targets`

```yaml
target-path: "target"
clean-targets:
  - "target"
  - "dbt_packages"
```

`target-path` is where dbt writes its compiled SQL, run logs, and the manifest. After
any `dbt run` you'll see a `target/` folder appear — that's everything dbt knows about
the last build. You can peek inside (`target/compiled/` is especially useful for
debugging — it's the actual SQL dbt sent to the warehouse).

`clean-targets` is what `dbt clean` deletes. Both `target/` and `dbt_packages/` (where
third-party dbt packages get installed) should be gitignored.

### `models:` block

```yaml
models:
  my_first_project:
    example:
      +materialized: view
```

This is where you set **defaults** for your models. The structure mirrors your
folder layout. Reading top-to-bottom:

- `models:` — the start of model-level config.
- `my_first_project:` — your project name (matches `name:` above). This scope means
  "apply to all models in THIS project."
- `example:` — the `models/example/` subfolder. This narrows the scope further.
- `+materialized: view` — the `+` prefix means "this is a config." So: "all models
  in `models/example/` default to being materialized as views."

A more realistic version, after you delete the example folder and structure your
project properly, would look like:

```yaml
models:
  my_first_project:
    staging:
      +materialized: view     # staging defaults to view (cheap)
    marts:
      +materialized: table    # marts default to table (fast to query)
```

You can also override per-file using a `{{ config(...) }}` block at the top of a
model. Project-level defaults are nice because you don't have to repeat them in
every file.

## What `dbt_project.yml` is NOT for

- **Not for credentials.** Project IDs, passwords, API keys, etc. live in
  `profiles.yml` (next file), not here. `dbt_project.yml` is checked into git;
  `profiles.yml` isn't.
- **Not for runtime variables.** Use the `vars:` block (we'll see this later) or
  pass `--vars` on the command line.

## Common confusion

> "Why are there two YAML config files? Why not one?"

Because git. `dbt_project.yml` is project config — same on every machine, checked
into version control. `profiles.yml` is environment config — different on every
machine (your project ID isn't the same as your teammate's), and sometimes contains
secrets. Splitting them lets you safely git-commit the project config without
committing your credentials.

## Try this

Open `dbt_project.yml` in your editor and change `name:` to something else (e.g.,
`my_first_project_v2`). Save. Run `dbt parse`. You'll get:

```
Could not find profile named 'my_first_project_v2'
```

That's because the `profile:` field still says `my_first_project` — and there's no
profile by that name in `profiles.yml`. Now you've seen the bridge in action.

**Revert the change** before continuing.

---

Now let's look at the other half of the bridge: `profiles.yml`.

← Previous&nbsp; [`05-dbt-init.md`](05-dbt-init.md) &nbsp;·&nbsp; Next →&nbsp; [`07-profiles-yml.md`](07-profiles-yml.md)
