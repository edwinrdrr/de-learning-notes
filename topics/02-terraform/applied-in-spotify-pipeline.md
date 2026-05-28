# Terraform — applied in `spotify-pipeline`

The two big things to look at:

## The module pair (`terraform/modules/`)

[`terraform/modules/data-project/main.tf`](https://github.com/edwinrdrr/spotify-pipeline/blob/main/terraform/modules/data-project/main.tf)
- The reusable env template: enabled APIs (with `deploy_function` flag that flips
  staging/prod-only APIs on), one GCS bucket with versioning, two BigQuery datasets,
  three service accounts (`dbt-ci` always; `spotify-ingest-fn` and `spotify-scheduler`
  conditional via `count = var.deploy_function ? 1 : 0`).
- **Pattern to learn**: `for_each = toset(local.apis)` to create one resource per
  string in a list — cleaner than `count` because keys are stable across re-orderings.

[`terraform/modules/wif/main.tf`](https://github.com/edwinrdrr/spotify-pipeline/blob/main/terraform/modules/wif/main.tf)
- WIF pool + provider in one shot. The `attribute_condition` pins on
  `assertion.repository_id` (numeric, immutable — survives repo renames).
- The impersonation binding uses
  `principalSet://iam.googleapis.com/.../attribute.repository/<repo>` — that string
  format took the project's author multiple tries to get right.

## The env folders (`terraform/envs/`)

[`terraform/envs/dev/main.tf`](https://github.com/edwinrdrr/spotify-pipeline/blob/main/terraform/envs/dev/main.tf)
- 30 lines. Calls the `data-project` module with `deploy_function = false`.
  Multi-line variable blocks (no `;` separators!).

[`terraform/envs/infra/main.tf`](https://github.com/edwinrdrr/spotify-pipeline/blob/main/terraform/envs/infra/main.tf)
- The Big One: the tfstate bucket (imported on first apply via
  `terraform import`), the ci-state bucket, the `tf-runner` SA with `editor` +
  `projectIamAdmin` on every project, the WIF module call (with `depends_on` to
  force ordering).
- **Pattern**: the `for_each` over `local.tf_runner_projects` creates an IAM binding
  per project — same SA, multiple projects.

[`terraform/envs/{dev,staging,prod,infra}/backend.tf`](https://github.com/edwinrdrr/spotify-pipeline/blob/main/terraform/envs/dev/backend.tf)
- Each env's state lives at its own prefix in the same tfstate bucket:
  `gs://<infra>-tfstate/envs/<env>/`. One bucket, one prefix per env, perfect isolation.

## The provisioning script

[`bootstrap.sh`](https://github.com/edwinrdrr/spotify-pipeline/blob/main/bootstrap.sh)
- 8 steps. Steps 1–6 use `gcloud` directly to create projects, link billing, enable
  APIs, create the tfstate bucket (chicken-and-egg: Terraform can't manage the bucket
  it stores state in until that bucket exists, so bootstrap creates it manually, then
  imports it via `terraform import google_storage_bucket.tfstate ...` in step 8).
- Step 7 applies env data projects (dev/staging/prod) in any order — independent.
- Step 8 applies infra LAST (creates WIF, binds the env SAs to GitHub).

## What to grep for

- `for_each` — used in many places for "N copies of resource X" patterns
- `depends_on` — used 3 times to force ordering when Terraform can't infer
- `import` block — `bootstrap.sh` runs `terraform import` for the tfstate bucket
- `user_project_override` — provider config that fixes
  `google_billing_budget` 403 errors

## What's NOT in spotify-pipeline (skip on first read)

- Custom providers / Provider configuration aliases
- Dynamic blocks (only used in 1 place, for budget thresholds — straightforward)
- Workspaces (envs/ folder structure replaces them — see Terraform Best Practices)
- Sentinel / OPA policy-as-code
