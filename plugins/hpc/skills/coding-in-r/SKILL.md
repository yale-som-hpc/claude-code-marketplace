---
name: coding-in-r
description: How to write research R — renv, tidyverse/data.table, project paths, style, scripts. TRIGGER when authoring or editing .R/.Rmd/.qmd files for research work. For running R on the Yale SOM HPC cluster (Slurm, renv on /gpfs), use running-r instead.
related:
  - programming-and-coding
  - running-r
  - working-with-large-data
  - code-review
updated: 2026-05-22
---
# Coding in R

Rule: `renv`, portable paths, script as source of truth. Never rely on the global workspace.

For cluster execution, also load [running R](../running-r/SKILL.md).

## Defaults

- `renv` for package versions. Commit `renv.lock` and `.Rprofile`.
- On SOM HPC: `module load r`; no `rig` or `mise` default.
- `pak` for fast installs when available.
- `styler` for formatting.
- `lintr` for linting.
- `here::here()` for paths. No `setwd()` in tracked scripts.
- `readr::read_csv()` over base `read.csv()`.

```r
install.packages("renv")
renv::init()
renv::install(c("tidyverse", "data.table", "here", "fs", "styler", "lintr"))
renv::snapshot()
```

Restore with:

```r
renv::restore()
```

On HPC, restore once before arrays. Do not let many workers install packages concurrently.

## Layout

```text
project/
├── renv.lock
├── .Rprofile
├── README.md
├── scripts/       # entry points
├── R/             # shared helpers
├── notebooks/     # .Rmd / .qmd
├── data/raw/      # immutable; usually ignored
├── data/derived/  # rebuildable
└── results/       # figures/tables/logs
```

## Paths

Never commit `setwd()` or personal absolute paths.

```r
library(here)
counts_path <- here("data", "raw", "counts.csv")
counts <- readr::read_csv(counts_path)
```

For machine-specific roots, read an environment variable from `.Renviron` or the shell.

## Style

- `snake_case` for variables/functions.
- `<-` for assignment; `=` for arguments.
- Native pipe `|>` unless `%>%` materially helps.
- 2-space indentation; aim for 80-char lines, max 120.
- Named arguments: `mean(x, na.rm = TRUE)`, not `mean(x, T)`.
- `library()` at top.
- Never `attach()` or `<<-`.
- `set.seed(42)` before stochastic work.

## Data

Default to tidyverse for clarity. Use data.table when data are large, joins/grouping dominate, or in-place mutation matters. Benchmark before rewriting clear code for speed.

## CLI skeleton

```r
#!/usr/bin/env Rscript
suppressPackageStartupMessages({
  library(optparse)
})

option_list <- list(
  make_option(c("-i", "--input"), type = "character"),
  make_option(c("-o", "--output"), type = "character")
)
opt <- parse_args(OptionParser(option_list = option_list))
if (is.null(opt$input) || is.null(opt$output)) {
  stop("--input and --output are required", call. = FALSE)
}

set.seed(42)
```

## Checklist

- [ ] `styler::style_dir(".")` run
- [ ] `lintr::lint_dir(".")` clean or justified
- [ ] No `setwd()` or personal paths
- [ ] `set.seed()` where needed
- [ ] `renv::snapshot()` if deps changed
- [ ] Small smoke test run
- [ ] HPC uses `module load r` + `renv`, not `rig`/`mise`

## Further reading

- [Tidyverse style guide](https://style.tidyverse.org/)
- [renv](https://rstudio.github.io/renv/)
- [data.table](https://rdatatable.gitlab.io/data.table/)
- [here](https://here.r-lib.org/)
