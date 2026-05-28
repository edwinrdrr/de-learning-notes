# 05 — `dbt init` — create your first project

← Previous&nbsp; [`04-installing-dbt.md`](04-installing-dbt.md) &nbsp;·&nbsp; Next →&nbsp; [`06-dbt-project-yml.md`](06-dbt-project-yml.md)

---

Your venv has dbt. Now let's create an actual dbt project.

## What is "a dbt project"?

A dbt project is just **a folder** with a particular structure:

- A `dbt_project.yml` file (we'll explore this in file 06).
- A `models/` subfolder for your SQL files.
- A few other subfolders: `seeds/`, `tests/`, `macros/`, `snapshots/`, `analyses/`.

That's it. dbt doesn't care if the folder is in git or not, if it's nested
somewhere, or what name you give it. The structure is the contract.

You could create this folder by hand — make the directories, write a
`dbt_project.yml` from scratch. But dbt has a command that does it for you:
`dbt init`. We'll use that.

## Step 1: Run `dbt init`

From your working folder (`~/learning/dbt-tutorial` with venv active):

```bash
dbt init my_first_project
```

`my_first_project` is the name we want for the project. dbt will create a folder by
that name in the current directory.

The command runs an **interactive wizard** that asks you a few questions to fill in
`profiles.yml` for you. Let's walk through them.

## Step 2: Answer the prompts

You'll see a series of prompts. Type the answers exactly as shown below.

### Which database would you like to use?

```
[1] bigquery
[2] postgres
[3] snowflake
...
```

(Numbers may differ depending on which adapters you've installed; with only
dbt-bigquery installed, you might only see one option.)

Type the number next to `bigquery` and press Enter.

### [1] oauth, [2] service_account

Type `1` for `oauth`. This corresponds to using ADC — the credentials you set up in
file 02.

### project (GCP project id)

**Paste your GCP project ID** from file 01 here. Mine looks like
`sweet-mountain-471812`. Yours is different.

Don't add quotes; just paste the string. Enter.

### dataset (the name of your dbt schema)

Type `dbt_tutorial`. This is what BigQuery will call the dataset (= schema) where
dbt creates its tables and views.

### threads (1 or more)

Type `4`. This is how many models dbt can build in parallel. For our tiny project
it doesn't matter; 4 is a reasonable default.

### job_execution_timeout_seconds

Press Enter to accept the default (300, = 5 minutes). Each dbt model that runs
longer than this will be killed. 5 minutes is more than enough for anything in
this tutorial.

### Desired location

Type `US`. This is the BigQuery dataset location. Matches the BigQuery sandbox
default.

### Done!

You'll see:

```
Profile my_first_project written to /home/you/.dbt/profiles.yml using project's profile_template.yml file and your supplied values. Run 'dbt debug' to validate the connection.
Your new dbt project "my_first_project" was created!
...
Happy modeling!
```

## Step 3: Look at what got created

```bash
ls
# my_first_project  .venv  (and maybe others)
```

There's a new folder `my_first_project/`. Step inside:

```bash
cd my_first_project
ls
```

You'll see:

```
analyses
dbt_project.yml
macros
models
README.md
seeds
snapshots
tests
```

Plus possibly a `target/` folder if you've run anything (you haven't yet, so likely
not).

Let's walk through each one.

| Folder/file | Purpose |
|---|---|
| `dbt_project.yml` | **The most important file**. We'll dissect it in file 06. |
| `models/` | Where your `.sql` model files live. dbt scaffolded two example models inside `models/example/`. |
| `analyses/` | One-off SQL queries dbt compiles but doesn't execute. Empty for now. |
| `tests/` | Custom SQL tests (vs the built-in column tests we'll see). Empty for now. |
| `seeds/` | CSV files dbt should load as tables. Empty for now. |
| `macros/` | Reusable Jinja macros. Empty for now. |
| `snapshots/` | Slowly-changing-dimension snapshot definitions (advanced; we won't use these). Empty for now. |
| `README.md` | A boilerplate README that dbt scaffolded. You can read it but it's not essential. |

You'll work mostly in `models/`, `seeds/`, and `dbt_project.yml`.

## Step 4: Where dbt put your profile

dbt also created (or appended to) the profile file:

```bash
cat ~/.dbt/profiles.yml
```

You'll see:

```yaml
my_first_project:
  outputs:
    dev:
      type: bigquery
      method: oauth
      project: sweet-mountain-471812
      dataset: dbt_tutorial
      threads: 4
      timeout_seconds: 300
      location: US
  target: dev
```

(Your `project:` value matches whatever you typed.) Don't worry about
understanding it yet — file 07 covers it in detail.

## Step 5: Make sure you're inside the project

For everything from here on, you need to be **inside** the project folder. Confirm:

```bash
pwd
# should end with /dbt-tutorial/my_first_project
```

If not, `cd my_first_project`.

## What `dbt init` actually did

To summarize:

1. Created a folder `my_first_project/`.
2. Populated it with the standard subfolder structure.
3. Wrote a `dbt_project.yml` with the project name baked in.
4. Wrote two example models in `models/example/` so you can run something
   immediately.
5. Asked you connection questions and wrote them into
   `~/.dbt/profiles.yml` so dbt knows how to reach BigQuery.

That's it. You could have done all this manually; `dbt init` is a convenience.

## What did you just do?

- Created your first dbt project (`my_first_project/`).
- Configured a connection to your BigQuery sandbox (`~/.dbt/profiles.yml`).
- Got familiar with the standard dbt folder layout.

## Common confusion

> "I accidentally hit Enter without typing my project ID. What now?"

The prompt probably defaulted to empty string or something. Delete the project
folder and re-run:

```bash
cd ~/learning/dbt-tutorial
rm -rf my_first_project
dbt init my_first_project
```

You can also just hand-edit `~/.dbt/profiles.yml` to fix the wrong values.

> "Can I rename the project later?"

You'd need to update `name:` and `profile:` in `dbt_project.yml`, rename the
folder, and update `~/.dbt/profiles.yml` (change the top-level key to match). Easier
to start over with `dbt init` and the right name.

> "What if I want the profile in the project folder, not `~/.dbt/`?"

You can. Move (or copy) `~/.dbt/profiles.yml` into your project folder, then:

```bash
export DBT_PROFILES_DIR=$PWD
```

dbt will use the in-folder version. This is the convention real projects use. For
this tutorial we stick with `~/.dbt/` for simplicity.

---

Time to slow down and actually understand the two config files dbt just created.

← Previous&nbsp; [`04-installing-dbt.md`](04-installing-dbt.md) &nbsp;·&nbsp; Next →&nbsp; [`06-dbt-project-yml.md`](06-dbt-project-yml.md)
