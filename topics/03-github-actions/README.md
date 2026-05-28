# GitHub Actions — placeholder

Not yet fleshed out. Priority is **01-dbt** while Edwin focuses on dbt learning;
GitHub Actions notes will follow in the same shape (`concepts.md`, `cheatsheet.md`,
`applied-in-spotify-pipeline.md`, and possibly `tutorial.md`).

When ready, the topic should cover:
- What CI/CD is, what GitHub Actions specifically does
- The workflow/job/step mental model
- Triggers (`pull_request`, `push`, `workflow_dispatch`, `schedule`)
- Matrix strategies
- `environment:` key + protection rules (required reviewer)
- Secrets vs Variables, repo-scope vs env-scope
- The OIDC token (`id-token: write` permission) for keyless cloud auth
- `paths:` filter — why doc-only PRs shouldn't fire deploys

`applied-in-spotify-pipeline.md` should point at:
- `.github/workflows/dbt-ci.yml` (matrix per env, WIF auth)
- `.github/workflows/dbt-deploy-prod.yml` (`environment: production` + manifest republish)
- `.github/workflows/terraform-ci.yml` (matrix + plan-on-PR + comment upsert)
- `.github/workflows/dbt-ci-cleanup.yml` (event = `pull_request: closed`)
