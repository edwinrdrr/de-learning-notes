# 01 — Get a free BigQuery sandbox

← Previous&nbsp; [`00-what-youll-build.md`](00-what-youll-build.md) &nbsp;·&nbsp; Next →&nbsp; [`02-gcloud-and-adc.md`](02-gcloud-and-adc.md)

---

Before dbt can do anything, it needs a database to talk to. We're going to use
**BigQuery** because (a) it has a free sandbox tier, no credit card required, and
(b) it's what `spotify-pipeline` uses, so what you learn transfers directly.

## What is BigQuery?

BigQuery is Google Cloud's **data warehouse**.

A data warehouse is just a database optimized for **analytics queries** —
"how many orders per customer last month" — instead of **transactional queries** —
"insert this new order." It can hold huge tables and run queries across billions of
rows in seconds. The downside: it's slower for tiny transactional work, so you
don't use it as the database behind a web app.

BigQuery is **serverless**: you don't pick a machine size or pay for "the database
running." You pay only for storage you use and queries you run. That's why a free
sandbox tier is possible — Google gives you a small allocation of both.

You'll see BigQuery sometimes called BQ for short.

## What is the "sandbox"?

The BigQuery sandbox is Google's free tier:

- **10 GB** of storage (we'll use a few KB)
- **1 TB** of queries per month (we'll use a few MB)
- No credit card required
- Tables auto-delete after 60 days (this won't affect us)
- Some features unavailable (streaming inserts, external tables — we don't need them)

For learning dbt, the sandbox is more than enough.

## Step 1: Open the BigQuery console

In your browser, go to: **https://console.cloud.google.com/bigquery**

Sign in with your Google account when prompted. A personal `@gmail.com` works
fine — you don't need a Google Workspace account.

## Step 2: Enable the sandbox

What happens next depends on whether you've used Google Cloud before:

### If this is your first time

You'll see a "Welcome to Google Cloud" or "Terms of Service" page. Accept the
ToS, pick your country, decline any marketing emails if you want, and continue.

Google may show you a popup like:

> **You've enabled the BigQuery Sandbox.**
> You can use BigQuery without billing. Some features are unavailable.

Click OK / Got it.

### If you've used Google Cloud before

You may already have a project set up. That's fine — you can use it. Look at the
top-left of the page for the project dropdown.

If you see "Activate" or a banner asking you to enable billing, **don't**. Just
close the banner. The sandbox works without billing.

## Step 3: Find your project ID

Look at the **top-left of the page**. You'll see a project selector. It might say
something like:

```
sweet-mountain-471812
```

That's your **project ID**. It's globally unique across all of Google Cloud — no
two projects share an ID. Yours will be different from mine.

**Click the dropdown to see it more clearly.** A popup appears showing your
projects. Your active one is the highlighted row. The ID is the string after the
project name (it might be the same as the name, or it might have a number suffix).

If you click on a project row in the popup, the "ID" column shows the exact string
you need.

**Write this ID down somewhere.** You'll paste it into multiple config files.

> ⚠️ Don't confuse the **project ID** (a string like `sweet-mountain-471812`) with
> the **project NAME** (a friendly label you picked, like "My First Project"). dbt
> needs the **ID**, not the name. The ID is the string with numbers/dashes at the end.

## Step 4: Click around (optional)

You're inside the BigQuery console. The left panel shows your project. If you
expand it, you'll see it's empty — no datasets, no tables. dbt will populate this
later.

The right panel is the query editor. You won't use it directly in this tutorial —
dbt will run queries for you — but you can come here to view tables dbt creates.

## What did you just do?

- You enabled **a free BigQuery sandbox** under your Google account.
- You got a **project ID** (which is a string GCP uses to identify your slice of
  the cloud).
- You confirmed the project exists by opening the BigQuery console.

You did NOT yet:
- Install anything locally.
- Connect dbt to BigQuery.
- Run any SQL.

Those come next.

## Common confusion

> "I see a billing prompt. Do I need to add a credit card?"

No. The sandbox works without billing. If you see "Activate," "Enable billing," or
"Add credit card," close those prompts. The free tier is what we want.

> "I already have a Google Cloud project from before. Can I use it?"

Yes. Use its project ID — write it down. The sandbox is per-account, not
per-project, so any project you can write to will work for this tutorial.

> "I'm getting an error like 'You don't have permission'."

Make sure you're signed into the right Google account (the one you just enabled the
sandbox under). If you have multiple Google accounts in the same browser, switch
profiles or use an incognito window.

---

You have a database. Now we need to give dbt the means to talk to it.

← Previous&nbsp; [`00-what-youll-build.md`](00-what-youll-build.md) &nbsp;·&nbsp; Next →&nbsp; [`02-gcloud-and-adc.md`](02-gcloud-and-adc.md)
