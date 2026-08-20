# 2. Running batches from an R script / CLI

Everything here was actually run and verified against the real API and the
real local executable during Packet 08 — this is the working version, not a
sketch.

## Setup

```r
pkgload::load_all(".", quiet = TRUE)   # dev checkout; use library(DDcentutils) if installed
```

```powershell
# one-time, per session, if not already persisted:
$env:DDCENT_SMOKE_EXE = [Environment]::GetEnvironmentVariable("DDCENT_SMOKE_EXE","User")
$env:DDCENT_SMOKE_100_DIR = [Environment]::GetEnvironmentVariable("DDCENT_SMOKE_100_DIR","User")
```

## A. One site/scenario, local executable

```r
config <- daycent_runner_config(
  "exe",
  exe_path = Sys.getenv("DDCENT_SMOKE_EXE"),
  dc_path100 = Sys.getenv("DDCENT_SMOKE_100_DIR"),
  validate_paths = TRUE
)

setwd("path/to/project/sites/wooster")
result <- DayCentRunSite(
  "wooster", "cc_nt",
  run_eq = FALSE,          # TRUE runs equilibrium + base first
  run_base = FALSE,
  backend = "exe",
  config = config
)
```

`run_eq = TRUE` also runs the base phase automatically for the local
backend; you'll see three phases print (`eq`, `base`, the scenario name)
instead of one.

## B. One site/scenario, API backend

```r
config <- daycent_runner_config(
  "api",
  base_url = "https://api.emdc.eco",
  model_name = "DayCent", model_version = "491"
  # api_key defaults to Sys.getenv("EMDC_API_KEY") — don't pass it literally
)

# 1. Dry run first — no credits spent, no run created.
preflight <- runDayCent_api(
  include = "pendleton/fw_fb_0n_blk6",
  run_eq = FALSE, run_base = FALSE,
  config = config, project_path = "path/to/project",
  name = "demo-dry-run", wait = FALSE,
  dry_run = TRUE, dry_run_mode = "Full"
)
preflight$passed          # must be TRUE before continuing
preflight$credit_cost     # inspect before spending anything

# 2. Real submission — only after reviewing the estimate above.
result <- runDayCent_api(
  include = "pendleton/fw_fb_0n_blk6",
  run_eq = FALSE, run_base = FALSE,
  config = config, project_path = "path/to/project",
  name = "demo-run", wait = TRUE      # waits for completion + downloads results
)
result$output_paths
```

## C. Batch: many pairs, local executable, parallel

```r
config <- daycent_runner_config(
  "exe", exe_path = Sys.getenv("DDCENT_SMOKE_EXE"),
  dc_path100 = Sys.getenv("DDCENT_SMOKE_100_DIR"), validate_paths = TRUE
)

# workers > 1 requires an injected executor that loads the package into
# each spawned worker process — pkgload::load_all() in your own session
# does NOT propagate to parallel::makePSOCKcluster() workers automatically.
parallel_executor <- function(tasks, worker, workers) {
  cluster <- parallel::makePSOCKcluster(workers)
  on.exit(parallel::stopCluster(cluster), add = TRUE)
  parallel::clusterCall(cluster, function(root) {
    pkgload::load_all(root, quiet = TRUE)
  }, root = ".")
  parallel::parLapply(cluster, seq_along(tasks), function(i, tasks, worker) {
    worker(tasks[[i]], i)
  }, tasks = tasks, worker = worker)
}

result <- runDayCent_batch(
  project_path = "path/to/project", backend = "exe", config = config,
  include = c("wooster/cb_nt", "wooster/cb_ct", "wooster/cc_nt", "wooster/cc_ct"),
  run_eq = FALSE, run_base = FALSE,
  workers = 2, executor = parallel_executor, error_policy = "collect"
)
result$task_table   # one row per site/scenario, with status
```

(If `workers = 1`, drop `executor` entirely — batches run sequentially in
your current session, no cluster needed.)

## D. Batch: all approved pairs, API, submit-then-resume

Useful for long-running batches you don't want to block a session on.

```r
config <- daycent_runner_config("api", model_name = "DayCent", model_version = "491")
pairs <- c("wooster/cb_nt", "wooster/cb_ct", "wooster/cc_nt", "wooster/cc_ct",
           "pendleton/fw_fb_0n_blk6", "pendleton/fw_nb_45n_blk1")

# Session 1: submit without waiting.
submission <- runDayCent_batch(
  project_path = "path/to/project", backend = "api", config = config,
  include = pairs, run_eq = FALSE, run_base = FALSE, wait = FALSE
)
parent_id <- submission$parent_run_id
# persist parent_id somewhere (a plain text file is fine — it's not a secret)
```

```r
# Session 2 (can be a genuinely fresh R session):
config <- daycent_runner_config("api", model_name = "DayCent", model_version = "491")
result <- runDayCent_batch(
  project_path = "path/to/project", backend = "api", config = config,
  include = pairs, run_eq = FALSE, run_base = FALSE,
  wait = TRUE, parent_run_id = parent_id   # resumes and downloads
)
result$task_table
```

Or let the server auto-discover every pair in the uploaded ZIP instead of
listing them explicitly:

```r
runDayCent_batch(project_path = "...", backend = "api", config = config,
                  include = "*", run_eq = FALSE, wait = TRUE)
```

## E. The ready-made CLI harness

For repeatable, gated execution (plan-only by default, explicit confirmation
tokens required before anything runs), `tests/smoke/run-daycent-runner-smoke.R`
wraps all of the above:

```powershell
# See the full resolved matrix, no execution:
Rscript tests/smoke/run-daycent-runner-smoke.R --plan-only

# One local case:
Rscript tests/smoke/run-daycent-runner-smoke.R --execute-local --case=atomic-local-scenario --confirm-local=RUN_LOCAL

# API: dry run, then submit, then (for the resume case) resume in a fresh session:
Rscript tests/smoke/run-daycent-runner-smoke.R --api-dry-run --case=atomic-api-scenario --confirm-api=DRY_RUN_atomic-api-scenario
Rscript tests/smoke/run-daycent-runner-smoke.R --execute-api --case=atomic-api-scenario --dry-run-evidence=<path from dry run> --credit-ceiling=1 --confirm-api=SUBMIT_atomic-api-scenario
```

See `tests/smoke/README.md` for the full command reference.

## Common failure you'll likely hit

If a batch with `workers > 1` fails with
`could not find function ".daycent_batch_row"` (or any other internal
package function), you forgot to load the package into the cluster workers —
see section C above.
