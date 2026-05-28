# 21 — `dbt build` — the one command you'll use most

← Previous&nbsp; [`20-breaking-tests.md`](20-breaking-tests.md) &nbsp;·&nbsp; Next →&nbsp; [`22-materializations.md`](22-materializations.md)

---

You've been running `dbt seed`, `dbt run`, and `dbt test` separately. In real life,
you almost always want all three in dependency order. That's what `dbt build` does.

Short file. ~5 minutes.

## What `dbt build` does

`dbt build` is `seed + run + test`, **in dependency order**, with one critical
extra rule:

> If a model's tests fail, the build does NOT continue to anything downstream
> of that model.

That's the magic. Bad data doesn't propagate. If `stg_customers` fails a test,
nothing that depends on `stg_customers` will be built. You stop and fix the problem
instead of unknowingly generating wrong results downstream.

In topological order, for each node:
1. Build the seed/model.
2. Run the tests for it.
3. If tests pass: proceed to nodes that depend on this one.
4. If tests fail: skip everything downstream.

## Step 1: Run it

```bash
dbt build
```

Expected (paraphrased):

```
Running with dbt=1.11.x
Found 3 models, 17 data tests, 2 seeds, ...

1 of 22 START seed file dbt_tutorial.raw_customers ........... [RUN]
2 of 22 START seed file dbt_tutorial.raw_orders ............... [RUN]
1 of 22 OK loaded seed file dbt_tutorial.raw_customers ........ [INSERT 10]
2 of 22 OK loaded seed file dbt_tutorial.raw_orders ............ [INSERT 10]
3 of 22 START sql view model dbt_tutorial.stg_customers ........ [RUN]
4 of 22 START sql view model dbt_tutorial.stg_orders ........... [RUN]
3 of 22 OK created sql view model dbt_tutorial.stg_customers .. [CREATE VIEW]
4 of 22 OK created sql view model dbt_tutorial.stg_orders ...... [CREATE VIEW]
5 of 22 START test ... [RUN]
6 of 22 START test ... [RUN]
...
17 of 22 PASS test ... [PASS in X.XXs]
18 of 22 START sql table model dbt_tutorial.dim_customers ...... [RUN]
18 of 22 OK created sql table model dbt_tutorial.dim_customers . [CREATE TABLE]
19 of 22 START test ... not_null_dim_customers_customer_id ..... [RUN]
...
22 of 22 PASS test ... [PASS in X.XXs]

Done. PASS=22 WARN=0 ERROR=0 SKIP=0 NO-OP=0 TOTAL=22
```

Read that order carefully:

1. Seeds built (`raw_customers`, `raw_orders`).
2. Staging models built (`stg_customers`, `stg_orders`) — they depend on the seeds.
3. Tests run **on staging**.
4. THEN the mart (`dim_customers`) builds — only because staging's tests passed.
5. Tests run on the mart.

If any staging test had failed, `dim_customers` would have been **skipped**, not
built.

## Step 2: See the skip behavior

Let's re-introduce the bad data from file 20 to see skipping:

1. Edit `seeds/raw_orders.csv`. Change one `completed` to `lost`.
2. Save.
3. Run `dbt build`.

Expected output (paraphrased):

```
... raw_orders loaded ...
... stg_orders built ...
... not_null_stg_orders_order_id PASS ...
... unique_stg_orders_order_id PASS ...
... accepted_values_stg_orders_status FAIL 1 ...
... dim_customers SKIP (parent failed) ...

Done. PASS=N WARN=0 ERROR=1 SKIP=4 NO-OP=0 TOTAL=22
```

Note `SKIP=4`:

- `dim_customers` was SKIPped because `stg_orders` had a failing test.
- The tests on `dim_customers` (which there are 3) were also SKIPped — they
  couldn't run against a table that wasn't built.

That's the whole point: **bad data is contained at the layer where it first
appears, not allowed to flow downstream.**

Revert the bad data, re-run `dbt build`, and you should see all 22 PASS.

## Step 3: The day-to-day workflow

You'll mostly use these three commands:

- **`dbt build`** — full project run + test. The default for any "I've made changes;
  rebuild" workflow.
- **`dbt build --select my_model+`** — rebuild a specific model and everything
  downstream. Tests included.
- **`dbt run --select my_model`** — quick model rebuild without tests, when
  iterating fast on a model's SQL.

`dbt seed`, `dbt run`, `dbt test` separately are mostly useful when you're
debugging a specific phase. For normal work: `dbt build`.

## How CI uses `dbt build`

In any continuous-integration setup (GitHub Actions, etc.), the workflow runs
`dbt build` on every PR. If anything — seed, model, or test — fails, the PR's
check fails. The PR can't merge until it's fixed.

That's the safety net. spotify-pipeline's `dbt-ci.yml` workflow runs `dbt build
--select state:modified.body+ --defer --state <prod-manifest>` for Slim CI — a
turbocharged version that only rebuilds changed models.

## What did you just do?

- Ran `dbt build` and saw it do seed → staging → test → mart → test in order.
- Watched the SKIP behavior when an upstream test failed.
- Learned the day-to-day workflow (`dbt build` is the default).

## Common confusion

> "`dbt build` is slower than `dbt run` because it also runs tests. Should I skip
> tests in dev?"

Don't. Tests are how you catch your own mistakes. Skipping them in dev means
you'll be the one introducing the bug. The few extra seconds are worth it.

> "What if I want `dbt build` but to NOT skip on test failure?"

There isn't a flag for that — skipping is the entire point of `dbt build`. If
you want "build everything regardless of tests," just run `dbt run` and then
`dbt test` separately. The tests will still report failures, but everything else
gets built.

> "Can I run a subset of models AND their tests?"

Yes — `--select` works on `dbt build`:

```bash
dbt build --select dim_customers
```

Builds dim_customers (and its upstream dependencies needed for it to compile),
runs the tests scoped to that scope.

> "Does `dbt build` rebuild seeds every time?"

Yes by default. If your seeds are big and slow, you can `--exclude resource_type:seed`:

```bash
dbt build --exclude resource_type:seed
```

For our tiny CSVs, just let them rebuild — it's a fraction of a second.

---

✅ **Part C checkpoint** — you understand tests, you've watched them catch bad
data, and you've used the one command (`dbt build`) that ties everything
together. The rest of the course adds polish.

← Previous&nbsp; [`20-breaking-tests.md`](20-breaking-tests.md) &nbsp;·&nbsp; Next →&nbsp; [`22-materializations.md`](22-materializations.md)
