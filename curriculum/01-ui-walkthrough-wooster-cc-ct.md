# 1. Running an example through the EMDC UI (wooster / cc_ct)

> **Note on this draft:** written without live access to the EMDC UI, so the
> steps below are a skeleton — the right *shape* of a point-and-click
> walkthrough, based on what the API contract underneath it requires (see
> `02-batch-cli-r-script.md` for the equivalent CLI calls). Replace each
> `[SCREENSHOT: ...]` marker with a real capture, and correct any menu
> labels/click order that don't match what you actually see.

## Goal

Run one site/scenario pair — `wooster` / `cc_ct` — through the UI end to
end: upload inputs, preflight, submit, watch status, download results.

## Prerequisites

Completed `00-setup-api-key-and-environment.md`: you're logged into the EMDC
UI with the same account whose API key you verified.

## Steps

1. **Prepare the input ZIP.**
   The UI needs the same `sites/wooster/...` layout the API expects: site
   files (`wooster_site.100`, `sitepar.in`, `soils.in`), the weather file(s)
   referenced by the schedule, and `wooster_cb_ct.sch` (plus `eq_outfiles.in`
   / `base_outfiles.in` / `scenario_outfiles.in` if you're running
   equilibrium). Zip the `sites/` folder at its top level.

   `[SCREENSHOT: Windows Explorer showing the sites/wooster folder about to be zipped]`

2. **Open the model run creation screen.**

   `[SCREENSHOT: EMDC dashboard, "New Model Run" or equivalent button]`

3. **Select the model identity.**
   Model name `DayCent`, version `491`. You should not need to supply a
   product ID directly — the platform resolves that internally from
   name+version.

   `[SCREENSHOT: model/version selector]`

4. **Upload the ZIP and set the run name.**
   Give it a descriptive, unique name (e.g. `wooster-cc_ct-demo`).

   `[SCREENSHOT: file upload + name field]`

5. **Set run options.**
   - `runEquilibrium`: off, for a scenario-only run of `cc_ct`.
   - Site/scenario selection: `wooster/cc_ct` (or leave as "all discovered"
     if the ZIP only contains this one pair).

   `[SCREENSHOT: run options panel]`

6. **Run preflight / dry run first.**
   Before submitting for real, use the platform's preflight/QC check (this
   is the same server-side check the CLI's `--api-dry-run` flag exercises —
   see doc 3 for what these findings mean and a real bug we hit here). Look
   for:
   - Overall pass/fail.
   - Estimated credit cost and task count.
   - Any QC findings (errors block the run; warnings don't).

   `[SCREENSHOT: preflight/QC results panel showing PASS and credit estimate]`

7. **Submit for real.**
   Only after preflight passes and you've confirmed the credit estimate is
   acceptable.

   `[SCREENSHOT: submit confirmation]`

8. **Watch status to completion.**
   Status should progress `Queued` → `Running` → `Completed` (or a terminal
   failure status — if so, stop and read the failure detail rather than
   resubmitting).

   `[SCREENSHOT: run status page showing Completed]`

9. **Download and inspect results.**
   Expect one output set per site/scenario: at minimum a `.bin`, a
   `_summary.out`, and `_dc_sip.csv` / harvest / year-summary files depending
   on your `outfiles.in` selection.

   `[SCREENSHOT: results download / file listing]`

## What "done" looks like

- Status: `Completed`.
- Downloaded ZIP contains `sites/wooster/outputs/cc_ct/...` (or platform's
  equivalent layout) with real output files, no empty/zero-byte archive.
- No error banners on the run detail page.

## Where this connects to the CLI

Everything above corresponds 1:1 to a `submit_daycent_run()` /
`runDayCent_api()` call with `dry_run = TRUE` (step 6) then `dry_run = NULL`
(step 7), `wait = TRUE` (step 8), and `download_daycent_results()` (step 9).
See `02-batch-cli-r-script.md` for the equivalent scripted version of this
exact same run.
