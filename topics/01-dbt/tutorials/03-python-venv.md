# 03 — Python virtualenv

← Previous&nbsp; [`02-gcloud-and-adc.md`](02-gcloud-and-adc.md) &nbsp;·&nbsp; Next →&nbsp; [`04-installing-dbt.md`](04-installing-dbt.md)

---

dbt is a Python package. Before we install it, we'll set up an **isolated Python
environment**, called a **virtualenv** (or "venv" for short). This file explains
what that is and why.

If you already know venvs, skim to "Step 1" and create one.

## What is a virtualenv and why?

Your computer probably has Python already installed at `/usr/bin/python3` (or
similar). Whenever you `pip install something`, Python adds that package to a
**global** location shared across all Python work on your machine.

That's fine for one or two tools, but it breaks down fast:

- Project A needs `requests==2.28`.
- Project B needs `requests==2.32`.
- You can only have one version of `requests` installed globally at a time.

A **virtualenv** is just a folder on disk that contains *its own copy* of Python and
*its own* `pip install` location. You "activate" it, install packages into it, and
those packages are visible only when that venv is active. Other projects don't see
them.

Mental model:
```
your system Python  ─→  global site-packages (shared, messy)
                                                  ↑
                                            avoid touching

your project venv   ─→  ./venv/lib/python3.x/site-packages (isolated)
                                                  ↑
                                          dbt lives here
```

When you activate the venv, your shell's `python` and `pip` commands point at the
venv's copies instead of the system ones. Deactivate, and they snap back to global.

This sounds like overhead but it's worth it. Two minutes of setup buys you years of
"my projects don't interfere with each other."

## Step 1: Pick a working directory

This is where everything lives — the venv, the dbt project, all the files.
Anywhere is fine. Make a folder:

```bash
mkdir -p ~/learning/dbt-tutorial
cd ~/learning/dbt-tutorial
```

> If you'd rather put it somewhere else, that's fine — just remember the path. The
> rest of the tutorial assumes you `cd`'d into this folder.

## Step 2: Create the venv

Python ships with a built-in venv creator:

```bash
python3 -m venv .venv
```

That does:
- Looks at your system Python 3.
- Creates a folder called `.venv` here.
- Inside `.venv`, copies (or symlinks to) the Python binary.
- Sets up `.venv/lib/python3.x/site-packages` for installing into.

Look at what got created:

```bash
ls .venv/
# bin  include  lib  lib64  pyvenv.cfg
```

The `bin/` folder is the most interesting one:

```bash
ls .venv/bin/
# activate  pip  pip3  python  python3  ...
```

Those `python` and `pip` are the venv's own copies. When activated, your shell
points at these.

## Step 3: Activate it

```bash
source .venv/bin/activate
```

Your prompt should change to show the venv name in parentheses:

```
(.venv) edwin@laptop:~/learning/dbt-tutorial$
```

That `(.venv)` is your visual confirmation that the venv is active. Until you
deactivate or close the terminal, `python` and `pip` now point at the venv.

Verify:

```bash
which python
# /home/edwin/learning/dbt-tutorial/.venv/bin/python

which pip
# /home/edwin/learning/dbt-tutorial/.venv/bin/pip
```

Both should point inside `.venv`. If they don't, the activation failed — re-run
`source .venv/bin/activate`.

## Step 4: Upgrade pip (optional but recommended)

```bash
pip install --upgrade pip
```

The pip that came with Python might be old. Upgrading takes 5 seconds and avoids
some "you have an outdated version" warnings later.

## Step 5: How to deactivate (for when you stop working)

When you're done for the day:

```bash
deactivate
```

Your prompt loses the `(.venv)` prefix, and `python` snaps back to the system
default. The venv folder is still there on disk; you'll re-activate next time:

```bash
cd ~/learning/dbt-tutorial
source .venv/bin/activate
```

> Activating a venv only affects the **current terminal session**. If you open a
> new tab, you have to activate again. This is normal.

## Step 6: Make `.venv` invisible to git (preview)

Later, when you turn this folder into a git repo, you'll want `.venv` ignored — the
folder is huge and machine-specific. The standard pattern is a `.gitignore` file:

```
.venv/
```

You won't make a git repo for this tutorial, but in real projects, that line is the
first thing to add.

## What did you just do?

- Created `~/learning/dbt-tutorial/` as your working folder.
- Created a virtualenv inside it at `.venv/`.
- Activated it — your terminal now uses an isolated Python.
- Upgraded pip.

You're ready to install dbt without polluting your system Python.

## Common confusion

> "Do I need a virtualenv? Can't I just `pip install dbt` globally?"

You can, but it'll bite you. dbt has many dependencies (Jinja2, pyyaml, requests,
google-cloud-bigquery...). When you have other Python projects, version conflicts
will eventually break something. The 2-minute venv investment pays back
immediately.

> "Why `.venv` and not `venv`?"

Convention. The leading dot makes the folder "hidden" in `ls` (so it doesn't
clutter the file listing) and it's the name editors like VS Code auto-detect for
"this folder contains a Python venv."

> "I'm on Windows and `source` doesn't work."

Use:
- PowerShell: `.venv\Scripts\Activate.ps1`
- cmd.exe: `.venv\Scripts\activate.bat`

The activation script exists; the syntax is just different.

> "After I close my terminal and reopen, my venv is gone."

The venv folder is still there on disk (`ls -la .venv/` to confirm). What's gone is
the *activation* — that lives in the shell session. Re-activate:

```bash
cd ~/learning/dbt-tutorial
source .venv/bin/activate
```

---

Clean Python environment, ready to receive dbt.

← Previous&nbsp; [`02-gcloud-and-adc.md`](02-gcloud-and-adc.md) &nbsp;·&nbsp; Next →&nbsp; [`04-installing-dbt.md`](04-installing-dbt.md)
