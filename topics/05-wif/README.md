# Workload Identity Federation — placeholder

Not yet fleshed out. Priority is **01-dbt** while Edwin focuses on dbt learning;
WIF notes will follow in the same shape.

When ready, the topic should cover:
- The pain WIF removes: long-lived SA key JSON files (no rotation, exfiltration risk)
- OIDC primer: what an ID token is, why federated identity matters
- The four-step exchange:
  1. GitHub mints an OIDC token (scoped to the workflow run)
  2. GCP STS exchanges it for an STS token (gated by attribute_condition)
  3. STS token impersonates a target SA (gated by `roles/iam.workloadIdentityUser`)
  4. SA's permissions are used to call GCP APIs
- `attribute_condition` — pin on `assertion.repository_id` (numeric, immutable),
  NOT `assertion.repository` (string, renameable)
- `attribute_mapping` — what's available to use in conditions (subject, repository,
  ref, environment, event_name, ...)
- `principalSet://` member syntax — the format that took multiple tries to get right

`applied-in-spotify-pipeline.md` should point at:
- `terraform/modules/wif/main.tf` — pool, provider, impersonation bindings
- `.github/workflows/dbt-ci.yml` — the `google-github-actions/auth@v2` stanza with
  `workload_identity_provider` + `service_account`
- IAM prerequisite: depends on 04-iam being understood first
