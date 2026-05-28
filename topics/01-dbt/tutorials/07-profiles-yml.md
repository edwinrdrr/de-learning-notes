# 07 — `profiles.yml`

← Previous&nbsp; [`06-dbt-project-yml.md`](06-dbt-project-yml.md) &nbsp;·&nbsp; Next →&nbsp; [`08-dbt-debug.md`](08-dbt-debug.md)

---

This is the file most beginners don't understand, including me when I first used
dbt. Let's slow down and actually figure out what it's for.

## Why dbt needs `profiles.yml` at all

dbt is a Python program running on your laptop. But the SQL it generates needs to
run *somewhere* — in a database. So dbt needs to know three things before it can
do anything useful:

1. **Which kind of database** (BigQuery? Snowflake? Postgres? Redshift?). They speak
   slightly different SQL and each needs its own Python library to talk to.
2. **Where that database lives** (for BigQuery: the GCP project ID; for others: a
   host, port, database name).
3. **How to authenticate** (which user, which credentials).

All of that lives in **`profiles.yml`**.

## Why it's a separate file (not in `dbt_project.yml`)

dbt could have put everything in one file. It chose to split for a real reason:

| `dbt_project.yml` | `profiles.yml` |
|---|---|
| Project code config | Connection config |
| **Same on every machine** | **Different on every machine** |
| No secrets — your model names aren't sensitive | Sometimes contains secrets (passwords, API keys) |
| **Goes in git** | **Does NOT go in git** |

If both lived in one file, you couldn't safely commit the project config to git
without also committing your credentials. The split lets the project travel (in
git) while the credentials stay local.

## Where it lives

By default, dbt looks at `~/.dbt/profiles.yml` — that is, `profiles.yml` inside the
`.dbt` folder in your home directory.

- `~` is shell shorthand for your home directory (`/home/yourname` on Linux,
  `/Users/yourname` on macOS, `C:\Users\yourname` on Windows).
- `.dbt` is a hidden folder (the leading dot makes it hidden in `ls`). dbt created
  it for you when you ran `dbt init`.

You can override the location with an environment variable:
```bash
export DBT_PROFILES_DIR=$PWD/dbt
```

This is the common pattern for real projects: keep `profiles.yml` *inside* the repo,
gitignored, so the project is more self-contained. We'll use the default
(`~/.dbt/profiles.yml`) for this tutorial because it's simpler.

## Open the file

In your terminal:
```bash
cat ~/.dbt/profiles.yml
```

(Or open it in your text editor.) You'll see something like:

```yaml
my_first_project:
  outputs:
    dev:
      type: bigquery
      method: oauth
      project: silver-spaceman-12345
      dataset: dbt_tutorial
      threads: 4
      location: US
  target: dev
```

Let's read it line by line.

## Line by line

### `my_first_project:`

The top-level key. This is the **profile name**. dbt uses it to match against the
`profile:` field in your `dbt_project.yml`. If they don't match, dbt errors with
`Could not find profile named '...'`.

Why "profiles" (plural) in the filename? Because you can have multiple. If you work
on three different dbt projects on the same laptop, your `profiles.yml` will have
three top-level keys — one per project:

```yaml
my_first_project:
  outputs: { ... }
analytics_at_work:
  outputs: { ... }
data_eng_homelab:
  outputs: { ... }
```

Each dbt project picks its own by name.

### `outputs:`

A *list* of named connections (places dbt can run against). Most projects have at
least two:

```yaml
outputs:
  dev:       # where I build and test while coding
    ...
  prod:      # the real thing
    ...
```

For now we only have `dev`.

### `dev:`

The name of one specific output. To run dbt against it: `dbt run --target dev`. (If
you omit `--target`, dbt uses whatever's named in `target:` at the bottom of the
profile — see below.)

You can call your outputs anything. Common names: `dev`, `prod`, `ci`, `staging`.

### `type: bigquery`

Tells dbt which kind of database this is. dbt-bigquery (the package you
pip-installed) is what handles `type: bigquery`. If you ever switch to Snowflake,
you'd pip-install `dbt-snowflake` and change this to `type: snowflake` (and rewrite
the rest of the fields accordingly — different databases need different config).

### `method: oauth`

For BigQuery specifically, this tells dbt **how to authenticate**. There are a few
options:

- `oauth` — use the **Application Default Credentials** you set up earlier with
  `gcloud auth application-default login`. **This is what we want.**
- `service-account` — use a service-account JSON key file at a given path.
- `service-account-json` — same but with the JSON inline in the profile.

We picked `oauth` because:
1. ADC is set up already.
2. No key files to manage.
3. No risk of accidentally committing a key.

### `project: silver-spaceman-12345`

**Your GCP project ID.** This is which BigQuery to write to. You wrote this down in
file 01.

If you get this wrong, dbt connects but errors when it tries to create datasets in
a project that doesn't exist (or that your user can't access).

### `dataset: dbt_tutorial`

The **BigQuery dataset** dbt will use. In BigQuery, "dataset" means roughly what
"schema" means in Postgres — a namespace inside a project that contains tables and
views.

If the dataset doesn't exist, dbt will create it the first time you run.

The full address of a BigQuery table is: `project.dataset.table`. So a model named
`stg_customers` will become:
```
silver-spaceman-12345.dbt_tutorial.stg_customers
```

### `threads: 4`

dbt can run independent models in parallel. `threads: 4` means up to 4 at a time.
For our tiny project this doesn't matter; for big projects (hundreds of models) it
matters a lot. `4` is a reasonable default.

### `location: US`

BigQuery datasets are tied to a geographic location. `US` is a multi-region (i.e.,
"somewhere in the United States, Google decides exactly which data center"). The
BigQuery sandbox uses `US` by default.

If you ever change this after datasets are created, you can't move existing datasets
— you'd need to create new ones in the new location.

### `target: dev`

The **default** output when you run `dbt run` (or any command) without `--target X`.

For now, since we only have `dev`, this is what we'll always use. Once you have
multiple outputs:
- `dbt run` → uses `target: dev` (the default)
- `dbt run --target prod` → uses the `prod` output (overrides the default)

## The mental model in 3 sentences

1. dbt looks at `profile:` in `dbt_project.yml`, gets a name.
2. dbt opens `profiles.yml`, finds that name, sees a list of outputs.
3. dbt picks the right output (`--target X` or the default), reads its connection
   details, and connects to your warehouse.

That's it. Everything else is just YAML formatting.

## Common confusion

> "Can I have multiple profiles in `profiles.yml`?"

Yes, one per project. The top-level keys are profile names. Your `dbt_project.yml`
picks one with `profile: ...`.

> "Can I have multiple outputs in one profile?"

Yes, that's the whole point. Real teams have at least `dev` and `prod`; many also
have `ci`, `staging`, etc. You switch with `--target NAME`.

> "What if I have secrets I don't want in a file?"

Use environment variables. dbt supports `{{ env_var('MY_SECRET') }}` in
`profiles.yml`, so you can write:

```yaml
project: "{{ env_var('GCP_PROJECT_DEV') }}"
```

Then `export GCP_PROJECT_DEV=silver-spaceman-12345` in your shell. dbt substitutes
at runtime. This is what spotify-pipeline does — see
`applied-in-spotify-pipeline.md`.

## Try this

Edit your `profiles.yml` and change `target: dev` to `target: nonexistent`. Save.
Run `dbt debug`. You'll see:

```
The profile 'my_first_project' does not have a target named 'nonexistent'
```

dbt tried to use the default target, looked for an output named `nonexistent`,
didn't find one, and complained. **Revert** before continuing.

---

You now know what both config files are for. Time to verify dbt can actually talk
to your BigQuery.

← Previous&nbsp; [`06-dbt-project-yml.md`](06-dbt-project-yml.md) &nbsp;·&nbsp; Next →&nbsp; [`08-dbt-debug.md`](08-dbt-debug.md)
