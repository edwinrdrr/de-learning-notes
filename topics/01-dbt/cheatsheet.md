# dbt — cheatsheet

## Setup

```bash
# install dbt + adapter (one per warehouse)
pip install dbt-bigquery        # also: dbt-snowflake, dbt-redshift, dbt-postgres

# init a new project
dbt init my_project

# tell dbt where profiles.yml lives (default ~/.dbt; project-local is cleaner)
export DBT_PROFILES_DIR=$PWD/dbt
```

## The four commands you'll use 99% of the time

```bash
dbt deps           # install packages declared in packages.yml
dbt build          # compile + run + test (DAG-aware)
dbt run            # compile + run, NO tests
dbt test           # only run tests on already-built models
```

## Other commands worth knowing

```bash
dbt compile                       # render Jinja, no warehouse call. Writes target/compiled/.
dbt parse                         # parse project (validate structure), no compile.
dbt seed                          # only load CSVs from seeds/.
dbt snapshot                      # run snapshots (SCD-2 capture).
dbt source freshness              # check source freshness vs declared SLA.
dbt clean                         # delete target/ and dbt_packages/ (build artifacts).
dbt list --resource-type model    # list all models dbt knows about.
dbt run-operation MACRO_NAME      # invoke a macro directly (e.g. ad-hoc Jinja-templated SQL).
dbt docs generate                 # build docs data (catalog.json).
dbt docs serve                    # local docs website on :8080.
```

## Selectors (`--select`)

```bash
dbt build --select stg_users                   # one model
dbt build --select staging                     # one folder
dbt build --select stg_users+                  # stg_users and everything downstream
dbt build --select +fct_orders                 # fct_orders and everything upstream
dbt build --select +fct_orders+                # both directions
dbt build --select state:modified.body+        # changed models + downstream (Slim CI)
dbt build --select tag:nightly                 # by tag set in model config
dbt build --exclude marts                      # everything except marts
```

## Targets and defer (Slim CI)

```bash
dbt build --target ci                                       # use the `ci` output in profiles.yml
dbt build --select state:modified.body+ \
          --defer --state ./prod-manifest                   # only build changed; missing → read from prod
```

## profiles.yml shape (BigQuery, OAuth/ADC)

```yaml
my_project:
  outputs:
    dev:
      type: bigquery
      method: oauth                       # uses ADC; for SA-key: method: service-account
      project: "{{ env_var('GCP_PROJECT_DEV') }}"
      dataset: my_project_dev
      location: US
      threads: 4
      timeout_seconds: 300
  target: dev
```

## dbt_project.yml shape

```yaml
name: 'my_project'
version: '1.0.0'
config-version: 2
profile: 'my_project'

model-paths: ["models"]
target-path: "target"
clean-targets: ["target", "dbt_packages"]

# Materialization defaults per folder
models:
  my_project:
    staging:
      +materialized: view
    marts:
      +materialized: table
```

## Sources (in `models/.../_sources.yml`)

```yaml
version: 2
sources:
  - name: raw
    database: "{{ target.project }}"   # or hardcoded project name
    schema: raw                        # = BigQuery dataset
    tables:
      - name: users
        columns:
          - name: id
            tests: [not_null, unique]
```

Reference in SQL: `{{ source('raw', 'users') }}`.

## Models (in `models/.../foo.sql`)

```sql
{{ config(materialized='table') }}    -- override the default

with src as (
    select * from {{ ref('stg_users') }}      -- reference another model
)
select id, count(*) as orders
from src
group by id
```

## Tests (in a `.yml` next to the model)

```yaml
version: 2
models:
  - name: fct_orders
    columns:
      - name: order_id
        tests:
          - not_null
          - unique
      - name: status
        tests:
          - accepted_values:
              arguments:
                values: ['new', 'shipped', 'delivered']
      - name: customer_id
        tests:
          - relationships:
              arguments:
                to: ref('dim_customers')
                field: id
```

## Incremental materialization

```sql
{{ config(
    materialized='incremental',
    unique_key='id',
    on_schema_change='append_new_columns'    -- otherwise new columns silently drop
) }}

select * from {{ ref('stg_events') }}
{% if is_incremental() %}
  where event_ts > (select max(event_ts) from {{ this }})
{% endif %}
```

`on_schema_change`: `ignore` (default, silent drop), `append_new_columns` (usually
right), `sync_all_columns` (aggressive), `fail` (safest for prod).

`dbt run --full-refresh` to rebuild from scratch.

## BigQuery-specific: partition + cluster

```sql
{{ config(
    materialized='incremental',
    unique_key='order_id',
    partition_by={
        'field': 'order_date',
        'data_type': 'date',
        'granularity': 'day'        -- 'hour'|'day'|'month'|'year'
    },
    cluster_by=['customer_id']      -- list or single string
) }}
```

`partition_by` prunes scan cost by date range. `cluster_by` sorts within
partitions for cheap per-key filters. Together: the standard high-volume fact
table pattern.

## Custom `generate_schema_name` (macros/generate_schema_name.sql)

```sql
{% macro generate_schema_name(custom_schema_name, node) -%}
    {%- set default_schema = target.schema -%}
    {%- if target.name == 'prod' -%}
        {%- if custom_schema_name is none -%}
            {{ default_schema }}
        {%- else -%}
            {{ custom_schema_name | trim }}     {# no prefix in prod #}
        {%- endif -%}
    {%- else -%}
        {%- if custom_schema_name is none -%}
            {{ default_schema }}
        {%- else -%}
            {{ default_schema }}_{{ custom_schema_name | trim }}
        {%- endif -%}
    {%- endif -%}
{%- endmacro %}
```

Default dbt behavior: `{{ default_schema }}_{{ custom_schema_name }}`. Override
because you usually want clean names in prod and prefixed names in dev/ci.

## Common gotchas

- **`accepted_values: { values: [...] }` is deprecated** in dbt 1.10+. Use:
  ```yaml
  accepted_values:
    arguments:
      values: [...]
  ```
- **`{{ ref('x') }}` resolves at compile time**, not runtime. Quoting matters.
- **`--full-refresh`** rebuilds incremental models from scratch (drops + recreates).
- **`DBT_PROFILES_DIR` env var** beats `~/.dbt/profiles.yml`. Use it to put the profile
  in your repo (project-local, env-var-driven for secrets).
