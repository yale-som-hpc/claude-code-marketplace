---
name: coding-in-r
description: R implementation rules for research code — renv, tidyverse/data.table, here::here(), CLI scripts, style, seeds, and portable paths. TRIGGER when editing .R, .Rmd, or .qmd files; writing R analysis scripts; structuring an R project; or refactoring R code for Yale SOM research work on a laptop or the HPC cluster.
related:
  - programming-and-coding
  - running-r
  - working-with-large-data
  - code-review
updated: 2026-05-22
---
# Coding in R

Rule: lock packages with `renv`, write portable paths with `here::here()`, and treat scripts as the source of truth — never the global workspace.

For cluster execution, also load [running R](../running-r/SKILL.md). For general coding philosophy, also load [programming and coding](../programming-and-coding/SKILL.md).

## Tooling defaults

- **`renv`** for project-isolated package versions. `renv::init()` once, `renv::snapshot()` before committing. Commit `renv.lock` and `.Rprofile`.
- **Cluster R modules on SOM HPC.** On the cluster, use `module load r` for the R executable. Do not use `rig` or `mise` as the default R installer there.
- **`pak`** for fast package installs when available.
- **`styler`** for formatting: `styler::style_dir(".")`.
- **`lintr`** for linting with a project `.lintr`.
- **`here::here()`** for project-relative paths. Never `setwd()` in tracked scripts.
- **`readr::read_csv()`** over base `read.csv()` for better defaults.

Optional:

- **`testthat`** for reusable functions and formulas. For analysis scripts, smoke tests on small real data are often more valuable.
- **`optparse`** for command-line R scripts run by shell or Slurm.

## Project setup with renv

```r
install.packages("renv")
renv::init()
renv::install(c("tidyverse", "data.table", "here", "fs", "styler", "lintr"))
renv::snapshot()
```

Anyone re-creates the package library with:

```r
renv::restore()
```

On the HPC, run install/restore once before submitting large job arrays. Do not let many workers install packages concurrently.

## Project layout

```text
project/
├── renv.lock
├── .Rprofile
├── README.md
├── scripts/             # batch entry points
├── R/                   # shared helpers sourced by scripts/notebooks
├── notebooks/           # .Rmd / .qmd exploration and figures
├── data/raw/            # never edit; usually gitignored
├── data/derived/        # rebuildable intermediates
└── results/             # figures, tables, logs
```

Use an R package layout (`DESCRIPTION`, `R/`, `tests/`) only if helpers are shared across projects or need formal documentation.

## Portable paths

Never commit:

```r
setwd("/Users/alice/Desktop/project")
read_csv("/home/netid/data.csv")
readRDS("/gpfs/scratch60/netid/tmp/model.rds")
```

Use:

```r
library(here)
counts_path <- here("data", "raw", "counts.csv")
counts <- readr::read_csv(counts_path)
```

If a shared data root differs by machine, read it from `.Renviron` or an environment variable, not from a hardcoded personal path.

## R style

- `snake_case` for variables and functions.
- `<-` for assignment; reserve `=` for function arguments.
- Native pipe `|>` preferred unless `%>%` materially helps.
- 2-space indentation; spaces around operators; aim for 80-char lines, max 120.
- Named arguments for clarity: `mean(x, na.rm = TRUE)`, not `mean(x, T)`.
- Implicit return; use `return()` only for early exits.
- `library()` calls at the top of scripts.
- Never `attach()` or `<<-`.
- Set seeds before stochastic steps: `set.seed(42)`.

## Data manipulation

Default to **tidyverse** for clarity. Use **data.table** when:

- datasets are large and dplyr is proven slow,
- in-place modification matters,
- joins/grouping dominate runtime,
- the data.table expression is simpler than the tidyverse equivalent.

For multi-file tabular work, DuckDB, Arrow, Parquet, and Polars patterns live in [working with large data](../working-with-large-data/SKILL.md).

## CLI script pattern

```r
#!/usr/bin/env Rscript
suppressPackageStartupMessages({
  library(optparse)
  library(here)
})

option_list <- list(
  make_option(c("-i", "--input"), type = "character", help = "Input CSV"),
  make_option(c("-o", "--output"), type = "character", help = "Output RDS")
)

opt <- parse_args(OptionParser(option_list = option_list))
if (is.null(opt$input) || is.null(opt$output)) {
  stop("--input and --output are required", call. = FALSE)
}

set.seed(42)
```

## Checklist

- [ ] `styler::style_dir(".")` run
- [ ] `lintr::lint_dir(".")` clean or findings justified
- [ ] No `setwd()` or personal absolute paths
- [ ] `set.seed()` set before stochastic work
- [ ] `renv::snapshot()` run when dependencies changed
- [ ] Small realistic smoke test run
- [ ] On HPC, use `module load r` + `renv`, not `rig` or `mise`

## Further reading

- [Tidyverse style guide](https://style.tidyverse.org/)
- [R for Data Science](https://r4ds.hadley.nz/)
- [renv documentation](https://rstudio.github.io/renv/)
- [pak](https://pak.r-lib.org/)
- [data.table reference](https://rdatatable.gitlab.io/data.table/)
- [`here` documentation](https://here.r-lib.org/)
