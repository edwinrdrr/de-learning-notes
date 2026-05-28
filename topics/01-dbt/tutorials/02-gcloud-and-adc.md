# 02 — `gcloud` and Application Default Credentials

← Previous&nbsp; [`01-bigquery-sandbox.md`](01-bigquery-sandbox.md) &nbsp;·&nbsp; Next →&nbsp; [`03-python-venv.md`](03-python-venv.md)

---

You have a BigQuery sandbox in the browser. Now we need to give dbt — which runs on
your laptop, not in a browser — the ability to authenticate as you and talk to that
sandbox.

## What is `gcloud`?

`gcloud` is the **Google Cloud command-line interface**. It's a tool you install on
your machine that lets you talk to Google Cloud from your terminal. It can do
basically everything the console UI can do, just typed.

dbt doesn't strictly *need* gcloud installed — but the easiest way to set up
authentication for dbt's BigQuery adapter is `gcloud auth application-default
login`. So we install it.

## What is ADC?

**Application Default Credentials (ADC)** is Google's standard for "how a program
running on your machine proves it's you."

Here's the situation: dbt is a Python program. When it wants to query BigQuery, it
needs to send a request that proves "I am Edwin, I'm allowed to do this." Without
ADC, you'd have to bake your credentials into the Python program somehow — which is
either inconvenient (typing a password every run) or insecure (a key file Python
reads on startup).

ADC fixes this with a convention:

1. You run `gcloud auth application-default login` once.
2. gcloud opens a browser, you sign in with your Google account.
3. gcloud saves a credential file at
   `~/.config/gcloud/application_default_credentials.json`. (Don't ever commit this
   to git — it's effectively a password.)
4. Any Google Cloud client library — including dbt-bigquery — looks at the standard
   ADC location automatically. Your credentials "just work."

The credential file holds a refresh token. Each time dbt runs, it uses the refresh
token to get a short-lived access token. You log in once; it works for weeks.

## Step 1: Install `gcloud`

### Linux

```bash
cd ~
curl -sSL -o gcloud-cli.tar.gz \
  https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-cli-linux-x86_64.tar.gz
tar -xzf gcloud-cli.tar.gz && rm gcloud-cli.tar.gz
./google-cloud-sdk/install.sh --quiet --path-update=true
```

The installer will add `~/google-cloud-sdk/bin/` to your PATH. Reload your shell:

```bash
source ~/.bashrc      # if you use bash
# or:
source ~/.zshrc       # if you use zsh
```

### macOS (with Homebrew)

```bash
brew install --cask google-cloud-sdk
```

### Windows

Download the installer at https://cloud.google.com/sdk/docs/install#windows and
run it.

### Verify

```bash
gcloud --version
```

Expected first line:
```
Google Cloud SDK 5xx.x.x
```

If you see "command not found," your shell hasn't picked up the new PATH. Close and
reopen your terminal, or `source` the rc file again.

## Step 2: Authenticate gcloud (the CLI itself)

```bash
gcloud auth login
```

A browser tab opens. Sign in with the same Google account you used for BigQuery.
Grant the requested permissions, close the tab.

You can verify:
```bash
gcloud auth list
```

You should see your email with `*` next to it (= active account).

## Step 3: Authenticate ADC (what dbt actually uses)

`gcloud auth login` (step 2) authenticates the gcloud CLI. ADC is a **separate**
authentication for application code. dbt uses ADC, not the gcloud CLI's auth. So:

```bash
gcloud auth application-default login
```

A browser tab opens **again** (yes, similar to step 2). Sign in with the same
Google account. Grant the permissions, close the tab.

After this, you'll have a file at:
```
~/.config/gcloud/application_default_credentials.json
```

That's your ADC credential.

## Step 4: Tell ADC which project to bill API calls to

When dbt makes API calls to Google, Google needs to know which project to *bill* for
those calls. That's not necessarily the same as the project where your data lives —
though in our case they're the same.

You set this once:

```bash
gcloud auth application-default set-quota-project YOUR_PROJECT_ID
```

Replace `YOUR_PROJECT_ID` with the ID you wrote down in file 01.

You'll see:
```
Quota project "YOUR_PROJECT_ID" was added to ADC...
```

This isn't strictly required for the sandbox (which doesn't bill), but it
suppresses a warning that dbt would otherwise print on every run.

## Step 5: Verify

This command asks Google "give me an access token for the current user." If ADC is
set up correctly, it succeeds silently:

```bash
gcloud auth application-default print-access-token > /dev/null && echo "ADC OK"
```

Expected output:
```
ADC OK
```

If you see an error here, jump to file 25 (common errors). The two common ones are:
"ADC not found" (re-run step 3) or "quota project not set" (re-run step 4).

## Why two `auth login` commands?

This trips everyone up:

| Command | Authenticates | Used by |
|---|---|---|
| `gcloud auth login` | The `gcloud` CLI itself | When you type `gcloud projects list` etc. |
| `gcloud auth application-default login` | Application code | When dbt, Python libraries, Terraform, etc. talk to Google |

They're separate identities, even though it's the same Google account. dbt uses
ADC. The gcloud CLI uses its own auth. Both are tied to your Google account but
live in different files on your machine. You typically run both on a fresh setup.

## What did you just do?

- Installed the `gcloud` CLI on your laptop.
- Logged into Google twice — once for the CLI, once for ADC.
- Got a credential file (`application_default_credentials.json`) that dbt will find
  automatically.
- Set the quota project so Google knows which project to attribute API calls to.

## Common confusion

> "Can I skip the gcloud CLI install and just put a service-account key JSON in
> profiles.yml?"

You can, and that's what production systems do. But for local development, ADC is
strictly safer: no key file to lose, expires when you sign out of your Google
account, no risk of accidentally committing a key. For this tutorial, ADC.

> "My terminal says `gcloud: command not found` after install."

Your shell hasn't picked up the new PATH. Either:
1. Close the terminal and reopen it.
2. Or manually: `export PATH="$HOME/google-cloud-sdk/bin:$PATH"` and add that line
   to your `~/.bashrc` or `~/.zshrc`.

> "What is the difference between 'authenticate' and 'authorize'?"

**Authenticate**: prove who you are. ("I am edwin@gmail.com.")
**Authorize**: prove you're allowed to do something. ("This account is allowed to
read BigQuery datasets in project X.")

ADC handles authentication. Authorization is what the GCP IAM system controls — and
your sandbox user has full access to your sandbox project, so authorization is
automatic for now.

---

Authentication is in place. Now let's set up a Python environment so dbt has a
clean home.

← Previous&nbsp; [`01-bigquery-sandbox.md`](01-bigquery-sandbox.md) &nbsp;·&nbsp; Next →&nbsp; [`03-python-venv.md`](03-python-venv.md)
