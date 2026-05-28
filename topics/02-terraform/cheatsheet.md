# Terraform — cheatsheet

## Install (Linux)

```bash
curl -sSL -o tf.zip https://releases.hashicorp.com/terraform/1.9.8/terraform_1.9.8_linux_amd64.zip
mkdir -p ~/bin && unzip -oq tf.zip terraform -d ~/bin && rm tf.zip
export PATH="$HOME/bin:$PATH"
terraform version
```

> Pin to 1.9.8 or later — older versions hit `openpgp: key expired` errors with current Google provider.

## The four commands you'll use 99% of the time

```bash
terraform init      # download providers; configure backend. Run after cloning or backend change.
terraform plan      # preview changes. Read-only. Always run before apply.
terraform apply     # execute. Asks for confirmation; use -auto-approve in scripts.
terraform destroy   # tear it all down. RARELY what you want in prod.
```

## State management

```bash
terraform state list                                    # what does state know about?
terraform state show google_storage_bucket.my_bucket    # show one resource's attrs
terraform state mv google_x.foo google_x.bar            # rename in state without re-creating
terraform state rm google_x.foo                         # forget about a resource (it stays in cloud)
terraform refresh                                       # re-read all resources from cloud (deprecated; use plan -refresh-only)
terraform force-unlock <LOCK_ID>                        # bust a stale state lock (use carefully)
```

## Importing existing resources

```bash
# command-line (Terraform <1.5)
terraform import google_storage_bucket.my_bucket my-bucket-name

# import block (Terraform 1.5+, declarative)
import {
  to = google_storage_bucket.my_bucket
  id = "my-bucket-name"
}
resource "google_storage_bucket" "my_bucket" {
  ...
}
```

## Minimal `main.tf` (GCS bucket on GCP)

```hcl
terraform {
  required_version = ">= 1.5"
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = "my-project-id"
  region  = "us-central1"
}

resource "google_storage_bucket" "my_bucket" {
  name     = "my-bucket-name-unique-globally"
  location = "US"
  uniform_bucket_level_access = true
}
```

## Variables

```hcl
# variables.tf
variable "project_id" {
  type        = string
  description = "GCP project id"
}

variable "region" {
  type    = string
  default = "us-central1"
}

# usage
provider "google" {
  project = var.project_id
  region  = var.region
}
```

Pass values via `terraform.tfvars` (gitignored), `-var=foo=bar`, or `TF_VAR_foo=bar`
env var. **One-liner blocks are NOT valid HCL** — `variable "x" { type = string ; default = "y" }` errors. Use multi-line.

## Outputs

```hcl
output "bucket_url" {
  value = "gs://${google_storage_bucket.my_bucket.name}"
}
```

`terraform output bucket_url` (or `-raw bucket_url` for unquoted).

## `for_each` (avoid `count` when possible — `for_each` keys are stable)

```hcl
locals {
  apis = ["storage.googleapis.com", "bigquery.googleapis.com"]
}

resource "google_project_service" "apis" {
  for_each = toset(local.apis)
  project  = var.project_id
  service  = each.key
}
```

> Gotcha: `for_each` keys must be **known at plan time**. If you compute a key from a
> resource that doesn't exist yet, Terraform errors with "Invalid for_each argument."
> Use a static string interpolation instead.

## `depends_on` (force ordering)

Used when Terraform can't infer a dependency from references (e.g., a module call
where the inputs are static strings but you need the resource to exist first):

```hcl
module "wif" {
  source = "../modules/wif"
  ...
  depends_on = [google_service_account.tf_runner]
}
```

## Remote state (GCS backend)

```hcl
# backend.tf
terraform {
  backend "gcs" {
    bucket = "my-tfstate-bucket"
    prefix = "envs/dev"
  }
}
```

Then `terraform init` configures the backend. To migrate from local: `terraform init
-migrate-state`. Each `prefix` is independent state — perfect for one prefix per env.

## Modules

```hcl
# in envs/dev/main.tf
module "data" {
  source     = "../../modules/data-project"
  project_id = var.project_id
  env        = "dev"
}

# call its outputs
output "raw_bucket" {
  value = module.data.raw_bucket
}
```

## Common gotchas

- **`\$` is not a valid HCL escape** — write `$` literally in strings.
- **`;` is NOT an arg separator** inside a block. Use newlines.
- **`roles/editor` does NOT include `setIamPolicy`** on the project — to manage
  `google_project_iam_member`, also grant `roles/resourcemanager.projectIamAdmin`.
- **Plan with `-lock=false`** if running concurrent plans in CI. Plan is read-only.
- **State drift** = console-edited a resource Terraform manages. Either re-run
  `apply` (Terraform overwrites manual changes) or `terraform import` the new state.
- **`disable_on_destroy = false`** on `google_project_service` — leaves APIs enabled
  if you ever destroy. Otherwise destroy can cascade-break other projects.
- **Provider auth on GCP** uses ADC by default. For `google_billing_budget`, also set
  `user_project_override = true` and `billing_project = var.project_id` on the
  provider block.
