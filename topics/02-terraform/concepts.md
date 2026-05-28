# Terraform — concepts

## What is it?

**Terraform** is a tool that lets you write down *what infrastructure should look like*
in text files, then it figures out what to create, update, or delete to make reality
match the text. The text is checked into git; the infrastructure is reproduced from
the text by running one command.

This pattern has a name: **Infrastructure as Code** (IaC). You stop clicking around
in cloud consoles (that's "click-ops"); your infra becomes a versioned, reviewable,
diffable artifact like any other code.

## What problem does it solve?

Before IaC, infra was a mess:
- "How did we set up the production environment again?" → nobody knew. Sometimes the
  one person who did know had quit.
- New environment for a customer? Three days of clicking, with at least one missed
  checkbox that bit you in week two.
- Cloud bill exploded. Why? Someone manually spun up something for testing and forgot.
- Change review meant a Google Doc describing what someone was about to do — not a
  diff anyone could actually evaluate.

Terraform solves all four:
- "How did we set up prod?" → `git checkout main && terraform apply`. Reproduced.
- New environment → copy a folder, change two values, apply. Hours, not days.
- Cost transparency → every resource has a file, owned by a team, in a PR.
- Change review → `terraform plan` outputs a diff that anyone can read.

## The mental model

Terraform sits between you and the cloud provider's API. You describe **resources**
(buckets, VMs, IAM bindings) in a language called **HCL**. Terraform compares your
description to the **state** (a JSON snapshot of what it last knew was real), figures
out the difference, and calls cloud APIs to converge.

```
HCL (.tf files)  →  terraform plan   →  diff (add/change/destroy)
                                          ↓
                    terraform apply  →  cloud API calls  →  infra now matches HCL
                                          ↓
                                    terraform.tfstate (updated)
```

### Three things that always live together
- **The `.tf` files** (your description, in git)
- **The state file** (`terraform.tfstate` — Terraform's memory of "what's real")
- **The provider** (a plugin that talks to the actual cloud API: `hashicorp/google`,
  `hashicorp/aws`, etc.)

### Where state lives
- **Locally**: `terraform.tfstate` in the current dir. Fine for solo. Sad if you lose
  the laptop, dangerous if two people edit simultaneously.
- **Remotely** (a "backend"): GCS bucket, S3 bucket, Terraform Cloud. Shared,
  versioned, locked during operations. **Standard for any team.**

### Modules: code reuse
A `module` is a folder of `.tf` files you can call from another folder with parameters.
You write the bucket+dataset+IAM pattern once, then instantiate it for `dev`,
`staging`, `prod` with different inputs.

```
terraform/
├── modules/
│   └── data-project/     # the reusable template
└── envs/
    ├── dev/              # calls the module with project=...-dev
    ├── staging/
    └── prod/
```

## Key terms

| Term | Meaning |
|---|---|
| **provider** | A plugin that knows how to call a cloud's API. `provider "google" { ... }` configures the GCP provider. |
| **resource** | A thing in the cloud. `resource "google_storage_bucket" "my_bucket" { name = "..." }` declares one. |
| **data source** | A *read-only* lookup (no creation). `data "google_project" "this" { ... }` reads metadata. |
| **state** | Terraform's JSON record of which real resources correspond to which `.tf` blocks. `terraform.tfstate`. |
| **plan** | A preview of what `apply` would do. Read-only. |
| **apply** | Execute the plan. Changes real infra. |
| **destroy** | The opposite of apply — remove all the resources in state. |
| **backend** | Where the state file lives (local, GCS, S3, etc.). |
| **module** | A reusable folder of `.tf` called from another `.tf`. |
| **variable** | An input to a module or env. `variable "project_id" { type = string }`. |
| **output** | A value the module exports for callers to use. |
| **import** | Bring an existing cloud resource under Terraform management without recreating it. |
| **`for_each` / `count`** | Create N copies of a resource. |
| **`depends_on`** | Force ordering when Terraform can't infer it from references. |

## When not to use Terraform

- **One-off, never-again resources** — a script you'll run once. The TF setup cost is wasted.
- **Application data + configuration** — Terraform is for infrastructure (buckets,
  network, IAM), not for "rows in this table." Tools like Ansible/dbt own that.
- **Real-time mutations** — if a resource changes 100x/day, Terraform isn't the right
  tool (you'd run `apply` constantly). Use the cloud API directly.

## Common misuses

- **Manually editing state** — `terraform state` commands exist, but raw JSON edits
  break invariants. Use `terraform import`, `state mv`, `state rm`.
- **Mixing apply and click-ops** — once a resource is in Terraform, NEVER edit it from
  the console. The next `apply` will revert your manual change ("drift").
- **One giant `main.tf`** — splits as the project grows. By the time it's 500 lines,
  separate by domain (`network.tf`, `iam.tf`, `storage.tf`).
- **Hardcoding secrets in `.tf` files** — they end up in state and in git. Use
  `var.secret_value` from `.tfvars` (gitignored) or pull from a secret manager.
- **No remote state in a team** — two people running `apply` in parallel will corrupt
  local state. Use a GCS/S3 backend with locking.

## Recommended external reading

- [Terraform Tutorial — Get Started (GCP)](https://developer.hashicorp.com/terraform/tutorials/gcp-get-started) — 30 minutes, hands-on.
- [Terraform Best Practices](https://www.terraform-best-practices.com/) — community guide; the "code structure" chapter is gold.
- [Google Cloud provider docs](https://registry.terraform.io/providers/hashicorp/google/latest/docs) — the actual resource reference. Bookmark this.
