# IAM — placeholder

Not yet fleshed out. Priority is **01-dbt** while Edwin focuses on dbt learning;
IAM notes will follow in the same shape.

When ready, the topic should cover:
- Identities: user accounts vs service accounts (SAs)
- The role/binding model: principals + roles + resources
- Predefined roles (start here) vs custom roles (almost never needed)
- The "blast radius" mental model: role-on-resource at the narrowest scope
- ADC (Application Default Credentials) — what it actually is, how libraries find it
- The "editor vs owner vs projectIamAdmin" gotcha (`editor` doesn't include `setIamPolicy`)
- Propagation lag (~30s for SA creation, ~30s for IAM bindings)

`applied-in-spotify-pipeline.md` should point at:
- `terraform/modules/data-project/main.tf` — `google_project_iam_member` resources
- `terraform/envs/infra/main.tf` — tf-runner's roles incl. `roles/resourcemanager.projectIamAdmin`
- `deploy.sh` — `gcloud run services add-iam-policy-binding` for the run.invoker grant
- `JOURNEY.md` Phase 11 lessons — SA propagation lag, IAM propagation ~30s
