# 00 — What you'll build

Next →&nbsp; [`01-bigquery-sandbox.md`](01-bigquery-sandbox.md)

---

Before we start typing, a quick orientation: what is this course actually going to
produce, and why is it in this order?

## What you'll have at the end

A small data pipeline that runs against a real cloud data warehouse (BigQuery). It
won't be useful in the real world — it'll be a tiny example using customer + order
data — but it'll use **every fundamental dbt concept**.

Concretely, you'll have:

```
your laptop
│
├── a Python environment with dbt installed
│
└── a dbt project (a folder of SQL files + config) that does this:
    │
    │  raw CSVs ───► BigQuery tables (the "raw" layer)
    │       │
    │       ▼
    │  staging models ───► BigQuery views (light cleanup)
    │       │
    │       ▼
    │  one mart model ───► BigQuery table (analytics-ready, joined data)
    │       │
    │       ▼
    │  tests on every layer
    │       │
    │       ▼
    │  a documentation website you can browse locally
```

When you can confidently re-build this from scratch without looking at the tutorial,
you've reached "I can use dbt."

## What you'll know

After this course you'll understand:

- **What dbt is** and what it isn't (it's a SQL compiler with tests, not an
  orchestrator, not a data mover)
- **How dbt connects to BigQuery** (the `profiles.yml` file and what each line means)
- **What `dbt run`, `dbt test`, and `dbt build` actually do** under the hood
- **Seeds, sources, models, refs, materializations, tests** — the six core concepts
- **The layered architecture** (sources → staging → marts) and why it's a convention
- **How to read someone else's dbt project** (like `spotify-pipeline`'s)

## Why this order

Most dbt tutorials skip the setup and jump straight to writing models. That's fine if
you already have a database. But you don't — yet. So we'll do this:

1. **Part A: Get dbt to run.** This is the part where most beginners get stuck and
   give up. We'll set up a free BigQuery sandbox, install dbt, and verify the
   connection. **By end of Part A you've typed zero SQL** but the foundation is
   solid.

2. **Part B: First models.** Now SQL starts. We'll build a tiny pipeline from raw
   CSVs through staging to a mart. This is where the dbt mental model clicks.

3. **Part C: Tests.** Data tests are dbt's superpower. We'll add them, run them,
   then break one on purpose to see what failure looks like.

4. **Part D: Materializations + docs + common errors.** The "intermediate" stuff
   that turns "I built one project" into "I can build real ones."

## Prerequisites

You need:

- **A computer** with a terminal (Linux/macOS terminal, or Windows WSL/PowerShell).
- **A Google account** — for the free BigQuery sandbox. Doesn't need to be a Google
  Workspace account; a personal `@gmail.com` is fine.
- **Python 3.10 or newer** installed. To check: open your terminal and type
  `python3 --version`. If you see `Python 3.10.x` or higher, you're good.
- **About 3-4 hours** if you actually do everything, spread over a few sessions.

You do NOT need:

- Prior dbt experience.
- Prior BigQuery or cloud experience.
- A credit card (the BigQuery sandbox doesn't require one).
- Anything installed beyond Python — we'll install dbt and the gcloud CLI together.

## What about cost?

Everything in this course costs **$0**. The BigQuery sandbox is free with limits we
won't come close to (10 GB storage, 1 TB queries per month — we'll use kilobytes).

The only way you could accidentally incur cost is if you upgrade out of the sandbox
to "paid mode." We won't do that.

## How to use these files

- **One file = one concept.** They're short on purpose. You're meant to read,
  understand, sometimes type along, then move on.
- **Type the commands** — don't just read. The learning happens when your fingers
  do it, not when your eyes do it.
- **It's OK to pause mid-tutorial.** Each file is a logical stopping point. If you do
  half a session today and resume tomorrow, you won't lose context.
- **If something doesn't work**, jump to file 25 (common errors) before assuming
  you broke something. Often the error is one we've already documented.

---

Ready? Let's get a database.

Next →&nbsp; [`01-bigquery-sandbox.md`](01-bigquery-sandbox.md)
