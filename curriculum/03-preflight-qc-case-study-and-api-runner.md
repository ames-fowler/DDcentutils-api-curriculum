# 3. Reading preflight QC, a real bug we hit, and how the API runner works

## Part A — What preflight/dry-run QC actually returns

Every real submission should go through a dry run first (`dry_run = TRUE`).
It costs nothing and creates no model run — it's pure validation plus a
credit estimate.

```r
preflight <- runDayCent_api(
  include = "pendleton/fw_fb_0n_blk6", run_eq = FALSE, run_base = FALSE,
  config = config, project_path = "...", name = "dry-run",
  wait = FALSE, dry_run = TRUE, dry_run_mode = "Full"
)
str(preflight, max.level = 2)
```

Shape of the result:

```
$ dry_run           TRUE
$ passed             FALSE                # overall pass/fail
$ input_qc
  ..$ blocked        TRUE                 # any ERROR finding blocks the run
  ..$ counts         list(error=1, warn=0, info=0, total=1)
  ..$ findings       list of finding objects — see below
$ credit_cost
  ..$ credits        1
  ..$ totalExecutions 1
$ model_name / model_version / preflight_version
```

`passed` is only `TRUE` when `input_qc$counts$error == 0` — warnings don't
block. If `passed` is `FALSE`, don't submit; read the findings.

### Reading one finding

```r
preflight$input_qc$findings[[1]]
```

```
$ severity      "ERROR"                              # ERROR blocks; WARN doesn't
$ message       "Weather does not cover the scenario schedule year range."
$ file          "sites/pendleton/pendleton_fw_fb_0n_blk6.sch"
$ ruleCode      "SCH_WEATHER_COVERAGE_MISMATCH"       # stable identifier for this rule
$ confidence    "high"
$ context        # rule-specific structured detail — always worth printing in full
  ..$ scenario_start_year / scenario_end_year
  ..$ weather_start_year / weather_end_year
  ..$ weather_path
$ rule
  ..$ summary / detail / fixHint          # human-readable explanation + suggested fix
$ suggestedFix
```

Severity is not adjustable per-run — it's a property of the rule itself
(`ERROR` vs `WARN` vs `INFO`), fixed in the platform's rule registry, though
a few rules do escalate under stricter policies. Don't assume you can quiet
an `ERROR` by resubmitting; either the input needs to change or the rule
itself needs a fix (see Part B).

## Part B — Case study: `SCH_WEATHER_COVERAGE_MISMATCH` false positive

This is a real bug we hit and diagnosed end to end during Packet 08, worth
walking through because the diagnostic method generalizes.

**Symptom:** `pendleton/fw_fb_0n_blk6` failed dry-run preflight with
`SCH_WEATHER_COVERAGE_MISMATCH`, claiming weather didn't cover the schedule's
1931–1986 span.

**First instinct (wrong):** assume the staged weather file was the wrong
copy. Checked — the staged `pendleton1931.wth` was byte-identical to the
authoritative source file. Not a copy error.

**Second look — read the actual schedule, not just the error:**
`pendleton_fw_fb_0n_blk6.sch` has three internal blocks:

```
Block 1: 1931-1966, weather_choice F, file pendleton1931.wth (which only covers 1931-1934)
Block 2: 1967-1978, weather_choice C  (statistically generated — no file needed)
Block 3: 1979-1986, weather_choice C  (statistically generated — no file needed)
```

`C` blocks need no weather file at all. Only Block 1's short historical
record needs to be there, cycled — a completely standard DayCent technique
(confirmed: eq/base spin-ups elsewhere in this same codebase, e.g.
`wooster_eq.sch`, run a 33-year weather file across a 7000-year block the
same way).

**Third step — read the platform's own check code**, not just guess:
`check_scenario_weather_coverage()` in `emdc-platform` compares each
referenced weather file against the schedule's *whole-file* header span
(1931–1986), not the specific block that references it. It also, crucially,
is only invoked on the designated `scenario_schedule` file — `eq.sch`/
`base.sch` are exempt by design, which is *why* the exact same cyclic-weather
pattern never trips this rule in `wooster_eq.sch`. Pendleton's case is only
different because its eq-shaped block lives inside the scenario file itself
instead of a separate `eq.sch`.

**Confirmed, not assumed:** verified this exemption directly in
`project_qc.py` (`if sch_path == scenario_schedule:` gates the call), and
verified the "identical pattern, never checked" claim by reading
`wooster_eq.sch`'s actual header years and weather file coverage.

**Outcome:** this is a genuine upstream bug (in `emdc-platform`, not
DDcentutils) — write-up handed off separately, not fixed in this repo. As a
*local, scratch-only* workaround so testing could continue: materialized the
real 4-year record cycled end-over-end into a new file spanning the full
schedule, and repointed the schedule's weather-choice line at it. That's
real data mechanically repeated, not fabricated values — the same thing
DayCent already does internally for eq/base periods.

**Takeaway for next time you hit a QC error:** don't take the message at
face value or assume the fix is "get better data." Read the actual schedule
block structure, compare against a known-working example, and read the
check's own source before deciding whether it's your input or the checker
that's wrong.

## Part C — How the API runner pipeline fits together

```
daycent_runner_config()      → build config (base_url, model_name/version, api_key from env)
        ↓
zip_daycent_inputs()          → package sites/ into a ZIP (runDayCent_api does this for you)
        ↓
submit_daycent_run(dry_run=TRUE)  → preflight: QC + credit estimate, no run created
        ↓  (only if passed == TRUE and estimate is acceptable)
submit_daycent_run()          → real submission, returns run_id
        ↓
watch_daycent_run()            → polls status until a terminal state
        ↓
download_daycent_results()     → fetches + safely extracts the result ZIP
```

`runDayCent_api()` (single pair) and `runDayCent_batch()` (many pairs, one
parent + N children) both wrap this whole pipeline. The one thing worth
internalizing: **dry run and real submission are two separate, deliberate
calls** — nothing in this codebase auto-submits after a passing dry run. You
always decide, explicitly, whether the estimated cost is worth spending.
