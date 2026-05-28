# 23c — Multi-target: add a prod target, switch between

← Previous&nbsp; [`23b-packages.md`](23b-packages.md) &nbsp;·&nbsp; Next →&nbsp; [`23d-compiled-sql.md`](23d-compiled-sql.md)

---

So far you've only used the `dev` target. Every real project has at least two
targets — `dev` for building and testing, `prod` for the version stakeholders see.
Time to add a second target and try switching between them.

## What "multi-target" means in practice

Reminder from file 07: a **target** is one entry under `outputs:` in
`profiles.yml`. Each target has its own connection details (and often, its own
warehouse location).

Common targets:

| Target | Where it writes | Who triggers it |
|---|---|---|
| `dev` | Your personal dev dataset / project | You, while coding |
| `ci` | A CI-only dataset, ephemeral per PR | GitHub Actions on PRs |
| `staging` | A pre-prod dataset | CI on merge to main |
| `prod` | The real analytics dataset | Manual approve or scheduled cron |

The minimum useful setup is `dev` + `prod`. We'll add `prod`.

## Step 1: Add a `prod` output to `profiles.yml`

Open `~/.dbt/profiles.yml` in your editor. You should see:

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

Add a second output, `prod`. We'll point at the **same project** (we only have
one) but a **different dataset** (`dbt_tutorial_prod`):

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
    prod:
      type: bigquery
      method: oauth
      project: silver-spaceman-12345
      dataset: dbt_tutorial_prod    # ← different dataset
      threads: 4
      location: US
  target: dev
```

Save.

> In a **real** Level-3 setup, `dev` and `prod` would also point at different
> GCP projects entirely (not just different datasets). The principle is the same;
> the isolation is stronger.

## Step 2: Run with the `prod` target

```bash
dbt build --target prod
```

Expected output. Watch the dataset name in the model lines:

```
... raw_customers loaded into dbt_tutorial_prod.raw_customers ...
... stg_customers built into dbt_tutorial_prod.stg_customers (view) ...
... stg_orders built into dbt_tutorial_prod.stg_orders (view) ...
... dim_customers built into dbt_tutorial_prod.dim_customers (table) ...
```

Everything went into `dbt_tutorial_prod` instead of `dbt_tutorial`. dbt created
the new dataset automatically.

## Step 3: Verify in BigQuery

Open the BQ console. You should now see **two datasets**:
- `dbt_tutorial` — the dev build (from earlier files)
- `dbt_tutorial_prod` — the prod build (just now)

Both have the same models. Same data, since the source seeds are the same. The
isolation is in WHERE the tables live, not in WHAT they contain.

Try querying both:

```sql
-- dev
SELECT customer_id, total_spent
FROM `your-project.dbt_tutorial.dim_customers` ORDER BY total_spent DESC LIMIT 3
```

```sql
-- prod
SELECT customer_id, total_spent
FROM `your-project.dbt_tutorial_prod.dim_customers` ORDER BY total_spent DESC LIMIT 3
```

Identical results — same source seeds → same models. But the tables themselves
are different physical objects in different datasets.

## Step 4: The point of multi-target

Right now both targets show the same data because we used the same source seeds.
In a real project:

- **`dev`** would point at a small sample of data, or a snapshot from yesterday.
  Fast iteration. Cheap to rebuild.
- **`prod`** points at the real, full, fresh data. Slow, expensive to rebuild.
  You build this on a schedule (e.g., nightly via `dbt build --target prod` in a
  CI job).

When you change a model, you build against `dev` to verify it works on small
data. Then merge; CI builds against `prod` for the actual rollout. **Devs never
touch prod manually.**

## Step 5: Target-aware Jinja

dbt makes the current target available as a Jinja variable: `target`. You can use
it to write env-aware models.

Make a quick demo model. Create `models/staging/stg_target_info.sql`:

```bash
cat > models/staging/stg_target_info.sql <<'EOF'
-- This model just demonstrates target.* Jinja vars
select
    '{{ target.name }}'    as current_target,
    '{{ target.project }}' as current_project,
    '{{ target.dataset }}' as current_dataset
EOF
```

Run against both targets:

```bash
dbt run --select stg_target_info --target dev
dbt run --select stg_target_info --target prod
```

Query both in BQ:

```sql
SELECT * FROM `your-project.dbt_tutorial.stg_target_info`;
-- → current_target=dev, current_dataset=dbt_tutorial

SELECT * FROM `your-project.dbt_tutorial_prod.stg_target_info`;
-- → current_target=prod, current_dataset=dbt_tutorial_prod
```

Same model, different output based on which target it ran against.

Useful real-world example: only run sampling on dev, not on prod:

```sql
select * from {{ ref('large_table') }}
{% if target.name == 'dev' %}
  -- sample 1% in dev for fast iteration
  where rand() < 0.01
{% endif %}
```

Compiles to a full scan on prod, a 1% sample on dev. **One model file, env-aware
behavior.**

## Step 6: Switching the default target

`target: dev` at the bottom of `profiles.yml` makes `dev` the default. If you'd
rather have `prod` be the default (or just see what happens when you switch):

```yaml
  target: prod
```

Save. Then `dbt build` (no `--target`) builds against prod. Use this with caution
— it's easy to forget you switched and accidentally build into prod.

**Recommended**: leave `target: dev` as the default. Pass `--target prod` only
when you actually mean to.

## Cleanup

You can delete `stg_target_info.sql` and the `dbt_tutorial_prod` dataset:

```bash
rm models/staging/stg_target_info.sql
bq rm -r -f -d your-project:dbt_tutorial_prod
```

Or keep them — they don't hurt.

## What did you just do?

- Added a `prod` target to `profiles.yml` (different dataset).
- Built the project with `dbt build --target prod` — created a parallel set of
  tables in the prod dataset.
- Verified both dev and prod tables exist independently in BQ.
- Met the `target.*` Jinja variables for env-aware model logic.

## Common confusion

> "Do I need `target: dev` at the bottom of profiles.yml?"

It's the **default** when you don't pass `--target`. If you remove the `target:`
line, dbt errors with "no default target." Keep it.

> "Can I have more than 2 targets?"

Yes. Most real teams have `dev`, `ci`, `staging`, `prod` (sometimes more).
Define each one under `outputs:`. Switch with `--target NAME`.

> "Each target needs its own dataset?"

For BigQuery in single-project mode (like this tutorial), yes — that's the
isolation. In Level-3 setups, each target points at a different GCP project, and
the dataset can be the same name in each (since project is the env). The choice
is about how much isolation you want.

> "What if I want different SAs per target (production grade)?"

Add `method: service-account` + `keyfile: /path/to/key.json` per target. That's
how prod systems pin specific credentials per env. For local development, ADC is
fine.

> "Will `dbt build` against prod overwrite stuff?"

Yes — same as dev. Each `dbt build` does `CREATE OR REPLACE TABLE` for tables,
`CREATE OR REPLACE VIEW` for views. The previous version is gone. This is why
**only CI should target prod** — not humans.

---

You can now run the same project against multiple environments. Last piece: when
something goes wrong, look at the compiled SQL.

← Previous&nbsp; [`23b-packages.md`](23b-packages.md) &nbsp;·&nbsp; Next →&nbsp; [`23d-compiled-sql.md`](23d-compiled-sql.md)
