# 0. Setup: API key and local environment (Windows)

Do this once per machine before either the UI walkthrough or the CLI/batch
docs.

## 1. Get an EMDC API key

From the EMDC platform UI, generate/copy your personal API key (account or
API-settings area — exact menu path TODO, fill in from your own screenshots
since this doc was drafted without live UI access). Treat it like a password:
it grants your account's credit-consuming access.

## 2. Store the key in `.Renviron`, not in scripts

R reads `~/.Renviron` at every session startup and turns each line into an
environment variable, without you ever typing the key into a script or the
command line. On Windows, confirm which directory R treats as `~` first —
**this bit us during Packet 08**: if OneDrive is redirecting your `Documents`
folder, R's home is the OneDrive-redirected path, not `C:\Users\<you>`, and a
stray `.Renviron` sitting in the un-redirected location is silently ignored.

```r
# Run this in R/RStudio to find the file R actually reads:
Sys.getenv("HOME")
file.path(path.expand("~"), ".Renviron")
```

Open (or create) that exact file and add:

```
EMDC_API_KEY=your-real-key-here
EMDC_BACKEND_SERVER=https://api.emdc.eco
```

**Two mistakes that cost real debugging time on this project — check for both:**

- `EMDC_BACKEND_SERVER` **must include the scheme** (`https://`). A bare
  `api.emdc.eco` silently causes every request to go out schemeless, get
  302-redirected by the server, and fail in a way that looks like a backend
  bug but isn't.
- Make sure the key itself is current. A stale/revoked key doesn't always
  fail loudly — some read-only endpoints may still respond even with a bad
  key, while write endpoints return a bare `401` with no helpful body. Don't
  assume "this GET worked" means the key is good.

Save the file, then start a **fresh** R session (R only reads `.Renviron` at
startup) and verify:

```r
Sys.getenv("EMDC_API_KEY") != ""
Sys.getenv("EMDC_BACKEND_SERVER")
```

## 3. Verify the key works, read-only, no cost

From the DDcentutils repo root:

```powershell
Rscript tests/smoke/run-daycent-runner-smoke.R --api-auth-check
```

This makes exactly one read-only `GET` request and prints only the HTTP
status and a redacted redirect location — never the response body, never the
key. `HTTP status: 200` means you're set. Anything else, fix before moving on
to a dry run or a real submission.

## 4. Local executable backend (optional — only if you'll run cases 1/2 in
   the CLI doc locally instead of via the API)

Two things to point at:

- The licensed DayCent executable (e.g. `DDcentEVI_rev491.exe`).
- The `100_Files` **directory** (not a single file — this is a library
  directory of `.100` files passed to DayCent's `-l` option).

Set these once, persistently, as User environment variables:

```powershell
[Environment]::SetEnvironmentVariable("DDCENT_SMOKE_EXE", "C:\path\to\DDcentEVI_rev491.exe", "User")
[Environment]::SetEnvironmentVariable("DDCENT_SMOKE_100_DIR", "C:\path\to\100_Files", "User")
```

**Gotcha:** `[Environment]::SetEnvironmentVariable(..., "User")` writes to the
registry but does **not** update variables already loaded into a running
shell/session. Open a fresh terminal (or explicitly re-export both variables
at the start of each new PowerShell session) before running anything that
depends on them.

## 5. R package dependencies

The runner needs `httr`, `stringr`, and (for parallel/dev-mode work)
`pkgload` and `parallel`. If you're running from a source checkout rather
than an installed copy, dev-load the package each session rather than
`library(DDcentutils)`:

```r
pkgload::load_all(".", quiet = TRUE)
```

You're set up once: `--api-auth-check` returns `200`, and (if using the
local backend) both `DDCENT_SMOKE_EXE` and `DDCENT_SMOKE_100_DIR` point at
real, existing paths.
