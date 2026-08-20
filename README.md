# DDcentutils API Curriculum

Standalone fixtures and R Markdown walkthroughs for running DayCent through
the EMDC model-run API via [DDcentutils](https://github.com/CSU-Soil-Carbon-Solution-Center/DDcentutils/tree/daycent-runner-backends)
`runDayCent_api()` / `DayCentRunSite()` (`backend = "api"`). Split out from
`DDcentutils` so it can be shared and cloned independently of the main
package repo.

Everything here talks to the API — no local DayCent executable is required
to work through the curriculum.

## Prerequisites

- R with `DDcentutils` installed (or a dev checkout you can `pkgload::load_all()`)
- An EMDC platform account and API key (step 0 walks through getting one)

## Start here

Work through `curriculum/` in order:

1. [`00-setup-api-key-and-environment.md`](curriculum/00-setup-api-key-and-environment.md) — get an API key, set up `.Renviron`, get the DDcentutils API branch
2. [`01-ui-walkthrough-wooster-cc-ct.md`](curriculum/01-ui-walkthrough-wooster-cc-ct.md) — one site/scenario through the EMDC UI end to end
3. [`02-batch-cli-r-script.md`](curriculum/02-batch-cli-r-script.md) — running single and batch cases from an R script/CLI

## Fixtures

`fixtures/sites/` contains ready-to-submit DayCent site inputs, in the layout
the API expects, for two example sites referenced throughout the curriculum:

- `wooster/` — corn-corn / corn-bean, till/no-till schedule variants
- `pendleton/` — wheat-fallow schedule variants, multiple weather file options

These are the same fixtures the curriculum docs use in their example calls,
so you can follow along without assembling your own inputs first.
