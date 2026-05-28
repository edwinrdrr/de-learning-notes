# de-learning-notes

Data-engineering study notes. Companion to
[`edwinrdrr/spotify-pipeline`](https://github.com/edwinrdrr/spotify-pipeline) — that
repo is the *journey*, this repo is the *concepts*.

## How this is organized

One folder per topic under `topics/`. Each folder has up to **four files**:

| File | Purpose |
|---|---|
| `concepts.md` | **From-basic on-ramp.** Opens with "what is this and why does it exist," then mental model, key terms, when not to use it. Assumes only that you can use a terminal. |
| `cheatsheet.md` | Terse reference. Commands, syntax, config snippets — for when you know the topic but forgot the exact flag. |
| `applied-in-spotify-pipeline.md` | The bridge: how this topic shows up in spotify-pipeline, with file paths to look at. Keeps the notes anchored to a real codebase. |
| `tutorial.md` *(when present)* | Full step-by-step hands-on walkthrough. Copy-paste-friendly, common errors inlined, no "now go read the official docs." Takes ~2-3 hours to follow. |

## Topics

Numbered in a recommended study order (later topics build on earlier ones).

| # | Topic | Status | Files |
|---|-------|--------|-------|
| 01 | [dbt](topics/01-dbt/) — data transformation in SQL with tests | ✅ filled | concepts · cheatsheet · applied · **tutorial** |
| 02 | [terraform](topics/02-terraform/) — infrastructure as code | 🟡 mostly | concepts · cheatsheet · applied |
| 03 | [github-actions](topics/03-github-actions/) — CI/CD on GitHub | ⬜ stub only | placeholder |
| 04 | [iam](topics/04-iam/) — identities, roles, who can do what on GCP | ⬜ stub only | placeholder |
| 05 | [wif](topics/05-wif/) — Workload Identity Federation (keyless GitHub → GCP auth) | ⬜ stub only | placeholder |

✅ filled = useful end-to-end · 🟡 mostly = solid concepts + cheatsheet + applied · ⬜ stub only = roadmap of what should go in.

**Current focus is dbt.** Other topics will be filled in as study progresses.

Other topics to add later (the spotify-pipeline reproducer also needs working knowledge of these): bash, git, gh CLI, Python, SQL, REST APIs / OAuth, GCP fundamentals, GCS + BigQuery, Cloud Functions, data engineering concepts.

## How to use this repo

- **Studying from scratch?** Read `concepts.md` of each topic in order. Don't skip to applied yet.
- **Already know the topic, looking it up?** `cheatsheet.md` is your friend.
- **Reproducing spotify-pipeline?** Read `applied-in-spotify-pipeline.md` after the concepts to anchor what you just learned to a real file in a real repo.

## License

MIT.
