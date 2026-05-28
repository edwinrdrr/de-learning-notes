# dbt — applied in `spotify-pipeline`

How dbt shows up in the real codebase, in roughly the order you'd encounter it.

## The dbt project root

[`dbt/`](https://github.com/edwinrdrr/spotify-pipeline/tree/main/dbt) — the whole
project lives under one folder so it's clearly separated from the ingestion code.

## Configuration

[`dbt/dbt_project.yml`](https://github.com/edwinrdrr/spotify-pipeline/blob/main/dbt/dbt_project.yml)
- Single project (`spotify_pipeline`) — one profile, one model tree.
- Materialization defaults: `staging/` → `view`, `marts/` → `table`. **This is the
  layered-architecture convention from `concepts.md`.**

[`dbt/profiles.yml`](https://github.com/edwinrdrr/spotify-pipeline/blob/main/dbt/profiles.yml)
- **Three targets**: `dev`, `staging`, `prod`. Each reads `project:` from a different
  env var (`GCP_PROJECT_DEV`, `_STAGING`, `_PROD`) — same dbt code, different BigQuery
  project per env.
- `method: oauth` — uses ADC locally (`gcloud auth application-default login`) AND in
  CI (where `google-github-actions/auth@v2` populates ADC via WIF). Same auth path
  everywhere, no SA key JSON.
- Note `dataset: "{{ env_var('DBT_CI_DATASET', 'spotify_analytics') }}"` in the dev
  target — the CI workflow overrides this with `spotify_analytics_ci_pr_<n>` for
  per-PR ephemeral schemas.

## The model tree

[`dbt/models/staging/stg_top_tracks.sql`](https://github.com/edwinrdrr/spotify-pipeline/blob/main/dbt/models/staging/stg_top_tracks.sql)
- One model per source. Reads from `{{ source('spotify_raw', 'top_tracks') }}`,
  applies a `qualify row_number() = 1` dedupe. **Real-world pattern**: defensive
  dedupe in staging keeps marts correct even when upstream is messy.

[`dbt/models/staging/_spotify__sources.yml`](https://github.com/edwinrdrr/spotify-pipeline/blob/main/dbt/models/staging/_spotify__sources.yml)
- The `spotify_raw.top_tracks` source declaration + `not_null` tests on key columns.
  Source tests run on raw data — catch ingestion bugs before they reach analytics.

[`dbt/models/marts/fct_track_popularity_daily.sql`](https://github.com/edwinrdrr/spotify-pipeline/blob/main/dbt/models/marts/fct_track_popularity_daily.sql)
- The mart that answers "which tracks moved up / down / new since yesterday." Uses
  `lag(rank) over (partition by artist_track_key order by snapshot_date)` to compute
  day-over-day deltas. **Window functions are how analytics marts add the "compared to
  yesterday" column.**

[`dbt/models/marts/_spotify__marts_models.yml`](https://github.com/edwinrdrr/spotify-pipeline/blob/main/dbt/models/marts/_spotify__marts_models.yml)
- Model-level tests. Note `accepted_values` uses the **new `arguments:` nesting**
  (dbt 1.10+ syntax) — captured as a lesson in `JOURNEY.md` for Phase 5.

## CI integration (Slim CI)

[`.github/workflows/dbt-ci.yml`](https://github.com/edwinrdrr/spotify-pipeline/blob/main/.github/workflows/dbt-ci.yml)
- Look at the `dbt build` step — runs `--select state:modified.body+ --defer --state
  ../prod-state`. **Only rebuilds changed models + downstream**; everything else reads
  from the prod manifest dropped to `gs://<bucket>/dbt-state/manifest.json` by the
  prod workflow.
- `DBT_CI_DATASET: spotify_analytics_ci_pr_${{ github.event.pull_request.number }}` —
  the per-PR ephemeral schema name. Dropped by `dbt-ci-cleanup.yml` on PR close.

[`.github/workflows/dbt-deploy-prod.yml`](https://github.com/edwinrdrr/spotify-pipeline/blob/main/.github/workflows/dbt-deploy-prod.yml)
- The `Republish prod manifest` step — `gsutil cp dbt/target/manifest.json
  gs://<bucket>/dbt-state/manifest.json` — keeps the Slim CI baseline fresh.

## What's NOT in spotify-pipeline (yet, but you'll want for real projects)

- **`incremental` models** — `fct_track_popularity_daily` is currently a full rebuild
  every run. With more data, `materialized='incremental'` would skip rebuilding old
  snapshots.
- **Snapshots** — no SCD-2 tracking on `dim_artist` etc. (the project has only one
  artist dim and it's stable.)
- **dbt packages** (e.g., `dbt_utils`). The project keeps zero deps to stay self-contained.
- **dbt docs** (`dbt docs generate` + `dbt docs serve`). A real team would publish this.
