# dbt tutorials — index

A self-contained hands-on dbt course, broken into bite-sized files. **One concept
per file.** Read them in order; each one assumes you've read the ones before it.

Total time end-to-end: **~5–6 hours** if you actually type along.

Total time per file: **5–15 minutes**. So you can do a few, take a break, come back.

## Course outline

### Part A — get dbt working (no SQL yet)

| # | File | What you learn | Time |
|---|------|---------------|------|
| 00 | [`00-what-youll-build.md`](00-what-youll-build.md) | What this course produces. Why this order. | 5 min |
| 01 | [`01-bigquery-sandbox.md`](01-bigquery-sandbox.md) | What BigQuery is. Get a free sandbox. Find your project ID. | 15 min |
| 02 | [`02-gcloud-and-adc.md`](02-gcloud-and-adc.md) | The `gcloud` CLI and Application Default Credentials. | 10 min |
| 03 | [`03-python-venv.md`](03-python-venv.md) | What a virtualenv is and why we use one. | 5 min |
| 04 | [`04-installing-dbt.md`](04-installing-dbt.md) | Install dbt-bigquery. What pip actually installed. | 5 min |
| 05 | [`05-dbt-init.md`](05-dbt-init.md) | What `dbt init` does. The folders it scaffolds. | 10 min |
| 06 | [`06-dbt-project-yml.md`](06-dbt-project-yml.md) | The `dbt_project.yml` file. Project name, paths, materializations. | 10 min |
| 07 | [`07-profiles-yml.md`](07-profiles-yml.md) | The `profiles.yml` file. Why it's separate. What each line means. | 10 min |
| 08 | [`08-dbt-debug.md`](08-dbt-debug.md) | What `dbt debug` actually checks. How to read its output. | 5 min |

✅ **Checkpoint**: after Part A you have a dbt project that connects to BigQuery and a working brain about how the pieces fit together.

### Part B — your first models (SQL begins)

| # | File | What you learn | Time |
|---|------|---------------|------|
| 09 | [`09-the-example-models.md`](09-the-example-models.md) | Read the dbt-scaffolded examples. Understand them. Then delete them. | 10 min |
| 10 | [`10-dbt-run.md`](10-dbt-run.md) | What `dbt run` actually does, step by step. | 10 min |
| 11 | [`11-seeds.md`](11-seeds.md) | Seeds — loading CSVs as raw tables. | 10 min |
| 12 | [`12-jinja-and-ref.md`](12-jinja-and-ref.md) | Why dbt uses `{{ ... }}`. What `ref()` and `source()` are. | 10 min |
| 13 | [`13-first-staging-model.md`](13-first-staging-model.md) | Build your first staging model. | 15 min |
| 13b | [`13b-sources.md`](13b-sources.md) | The production pattern: declare a source, test the raw data, use `source()`. | 15 min |
| 14 | [`14-layered-architecture.md`](14-layered-architecture.md) | sources → staging → marts. Why three layers. | 10 min |
| 15 | [`15-first-mart-model.md`](15-first-mart-model.md) | Build a mart that joins two staging models. | 15 min |
| 16 | [`16-the-dag.md`](16-the-dag.md) | The dependency graph. How dbt figures out build order. | 10 min |

✅ **Checkpoint**: you have a real data pipeline (seeds → staging → marts) running in BigQuery.

### Part C — tests and safety

| # | File | What you learn | Time |
|---|------|---------------|------|
| 17 | [`17-why-tests.md`](17-why-tests.md) | What data tests are. Why dbt has them. | 5 min |
| 18 | [`18-column-tests.md`](18-column-tests.md) | `not_null`, `unique`, `accepted_values`, `relationships`. | 15 min |
| 18b | [`18b-custom-tests.md`](18b-custom-tests.md) | Singular custom tests in `tests/`. When built-ins aren't enough. | 10 min |
| 19 | [`19-running-tests.md`](19-running-tests.md) | `dbt test` — when and how to use it. | 5 min |
| 20 | [`20-breaking-tests.md`](20-breaking-tests.md) | Break a test on purpose. See what failure looks like. | 10 min |
| 21 | [`21-dbt-build.md`](21-dbt-build.md) | `dbt build` — the unified command. The one you'll actually use. | 5 min |

✅ **Checkpoint**: you can write data tests and trust them.

### Part D — go a little deeper

| # | File | What you learn | Time |
|---|------|---------------|------|
| 22 | [`22-materializations.md`](22-materializations.md) | view vs table vs incremental vs ephemeral. Plus BigQuery's partition_by/cluster_by. | 20 min |
| 22b | [`22b-incremental-built.md`](22b-incremental-built.md) | Actually build an incremental — watch MERGE, hit `on_schema_change`, use `--full-refresh`. | 20 min |
| 23 | [`23-configuring-mat.md`](23-configuring-mat.md) | How to set materialization per-model and per-folder. | 10 min |
| 23b | [`23b-packages.md`](23b-packages.md) | `packages.yml`, `dbt deps`, using `dbt_utils`. Plus a peek at macros. | 15 min |
| 23c | [`23c-multi-target.md`](23c-multi-target.md) | Add a `prod` target; switch with `--target`; env-aware `target.*` Jinja. | 10 min |
| 23d | [`23d-compiled-sql.md`](23d-compiled-sql.md) | `target/compiled/` — read it, debug with it. The dbt debugging superpower. | 10 min |
| 23e | [`23e-generate-schema-name.md`](23e-generate-schema-name.md) | Override `generate_schema_name` — the macro every real project rewrites. | 15 min |
| 24 | [`24-dbt-docs.md`](24-dbt-docs.md) | Generate documentation and serve it as a website. | 10 min |
| 25 | [`25-common-errors.md`](25-common-errors.md) | The errors you WILL hit, and what they actually mean. | reference |
| 26 | [`26-whats-next.md`](26-whats-next.md) | Where to go from here. Including the spotify-pipeline anchor. | 5 min |

✅ **Final checkpoint**: you can read spotify-pipeline's `dbt/` folder and follow it.

## Navigation

Each file has a "← Previous · Next →" line at the top and bottom. Just keep clicking
"Next." If you want to skip around, the index here is the table of contents.

## Status

✅ **Complete.** All 34 files written (27 main + 7 deep-dive `*b/c/d/e`). Total ~42k words; ~5-6 hours to follow.

The deep-dive files close gaps that the original 27 had:
- `13b-sources.md` — declaring sources (not just using seeds)
- `18b-custom-tests.md` — singular custom tests when built-ins aren't enough
- `22b-incremental-built.md` — actually building an incremental model
- `23b-packages.md` — `packages.yml`, `dbt deps`, `dbt_utils`, writing macros
- `23c-multi-target.md` — dev/prod targets, `target.*` Jinja
- `23d-compiled-sql.md` — debugging via `target/compiled/`
- `23e-generate-schema-name.md` — the macro every real project overrides

Plus BigQuery-specific configs (`partition_by`, `cluster_by`) appended to
file 22.
