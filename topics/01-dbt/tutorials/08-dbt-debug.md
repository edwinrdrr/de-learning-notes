# 08 — `dbt debug` — verify your setup

← Previous&nbsp; [`07-profiles-yml.md`](07-profiles-yml.md) &nbsp;·&nbsp; Next →&nbsp; [`09-the-example-models.md`](09-the-example-models.md)

---

Before running any models, we want to confirm dbt can actually connect to BigQuery.
That's what `dbt debug` is for. It's the "is everything wired up" check.

## What `dbt debug` checks

`dbt debug` runs through a checklist:

1. **Is `dbt_project.yml` in the current folder?** If you're not inside a dbt
   project, it errors here.
2. **Is `dbt_project.yml` valid YAML?** Syntax check.
3. **Can dbt find a profile by the name in `dbt_project.yml`'s `profile:` field?**
4. **Is the matching profile in `profiles.yml` syntactically valid?**
5. **Does the target output (`dev` by default) exist in that profile?**
6. **Can dbt establish a connection to the warehouse?** This is the big one —
   actually talks to BigQuery, runs a trivial query, makes sure auth works.

If any step fails, dbt tells you exactly which one and (usually) why.

## Step 1: Run it

From inside `my_first_project/`:

```bash
dbt debug
```

Expected output (paraphrased — exact text varies):

```
Running with dbt=1.11.x
dbt version: 1.11.x
python version: 3.11.x
python path: /home/edwin/learning/dbt-tutorial/.venv/bin/python
os info: Linux-...
Using profiles dir at /home/edwin/.dbt
Using profiles.yml file at /home/edwin/.dbt/profiles.yml
Using dbt_project.yml file at /home/edwin/learning/dbt-tutorial/my_first_project/dbt_project.yml

Configuration:
  profiles.yml file [OK found and valid]
  dbt_project.yml file [OK found and valid]

Required dependencies:
 - git [OK found]

Connection:
  method: oauth
  database: sweet-mountain-471812
  schema: dbt_tutorial
  location: US
  priority: interactive
  timeout_seconds: 300
  ...
  Registered adapter: bigquery=1.9.x
  Connection test: [OK connection ok]

All checks passed!
```

The two lines that matter most:

```
Connection test: [OK connection ok]
All checks passed!
```

If you see both, you're good. Move on.

## Step 2: If you see an error

Let's walk through the most common failures and what they actually mean.

### `No dbt_project.yml found at expected path ...`

You're not inside a dbt project. Either:
- `cd` into the project folder: `cd ~/learning/dbt-tutorial/my_first_project`.
- Or, you accidentally ran the command from your home directory or `.venv` parent.

### `Could not find profile named 'my_first_project'`

The `profile:` field in your `dbt_project.yml` doesn't match any top-level key in
`profiles.yml`. Two ways this happens:

- You renamed something in one file and forgot the other.
- `~/.dbt/profiles.yml` doesn't exist (dbt init didn't create it, or you moved it).

Check:

```bash
grep '^profile:' dbt_project.yml
# profile: my_first_project

head -1 ~/.dbt/profiles.yml
# my_first_project:
```

These two strings (after the colon, with leading whitespace stripped) must match
exactly.

### `BigQuery client configuration error: Cannot find credentials`

ADC isn't set up or has expired. Fix:

```bash
gcloud auth application-default login
```

(Same as file 02, step 3.)

### `403 ... User does not have permission ...`

Your Google account doesn't have access to the project ID in `profiles.yml`.
Either:
- You typed the wrong project ID. Re-check the BigQuery console (file 01) and edit
  `~/.dbt/profiles.yml`.
- You authenticated as a different Google account in step 2 (`gcloud auth
  application-default login`). Re-run that with the right account.

### `Quota project ... was not found`

ADC's quota project was set to something that doesn't exist (e.g., you deleted a
project earlier). Re-set it:

```bash
gcloud auth application-default set-quota-project YOUR_PROJECT_ID
```

### `Connection test: [FAIL Bad request: ... Not found: Dataset ...]`

This actually usually doesn't appear during `dbt debug` (which doesn't need the
dataset to exist), but if it does: the dataset doesn't exist yet AND your auth
isn't allowed to create it. Make sure you're using a project ID where you have
write access — your sandbox project should give you full access.

## Step 3: Understand `Connection test`

The `Connection test: [OK connection ok]` line means: dbt opened a connection to
BigQuery, ran a query that's roughly `SELECT 1`, got the right answer back, and
disconnected cleanly. It does NOT mean:

- Your dataset exists yet (it doesn't have to — dbt will create it).
- Your tables exist (we have no tables).
- Anything about your project's models being syntactically valid (dbt didn't read
  them).

`dbt debug` is purely about the dbt ↔ BigQuery handshake. Model validity is checked
later by `dbt parse` or `dbt run`.

## Why this command matters

Most setup pain happens *before* you write any SQL. `dbt debug` is your
hand-holder. If it says "All checks passed!", you can trust that anything that goes
wrong from here on is a real bug, not a setup problem.

Conversely: if you're ever debugging a weird issue (queries failing, models not
building), **run `dbt debug` first**. If it fails, your problem isn't your SQL —
it's your setup. Fix the setup first.

## What did you just do?

- Verified dbt can find your project.
- Verified your `profiles.yml` is reachable and valid.
- Verified ADC is in place and BigQuery accepts the credentials.
- Confirmed dbt can talk to BigQuery.

## Common confusion

> "What's the difference between `dbt debug` and `dbt parse`?"

| Command | Checks |
|---|---|
| `dbt debug` | Setup: config files, connection, credentials. |
| `dbt parse` | Project: are all `.sql` and `.yml` files syntactically valid? Does the DAG resolve? Doesn't talk to the warehouse. |
| `dbt run` | Both of the above + actually executes SQL against the warehouse. |

Run `dbt debug` after setup changes; `dbt parse` after model changes; `dbt run` to
actually do work.

> "Can I run `dbt debug` from somewhere other than the project folder?"

No. dbt looks for `dbt_project.yml` in the current directory (or a parent, walking
up). Without one, it errors out.

> "`dbt debug` succeeded but I'm worried I missed something."

Trust it. It's a complete check. If you make it through `dbt debug` clean, your
foundation is solid — you can move on confidently to writing SQL.

---

✅ **Part A checkpoint** — dbt is set up, the connection works, you understand
the config files. You're done with infrastructure. Time to actually use dbt.

← Previous&nbsp; [`07-profiles-yml.md`](07-profiles-yml.md) &nbsp;·&nbsp; Next →&nbsp; [`09-the-example-models.md`](09-the-example-models.md)
