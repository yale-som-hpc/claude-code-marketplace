---
name: running-r
description: Run R on the Yale SOM HPC cluster with Lmod modules, renv, batch scripts, and BLAS/OpenMP thread control. TRIGGER when writing R Slurm jobs on the Yale SOM HPC cluster, using renv on the cluster, installing R packages on the cluster, or running Rscript in batch mode there.
related:
  - installing-software
  - managing-jobs
  - using-the-filesystem
  - working-with-large-data
updated: 2026-04-29
---
# Running R

Rule: load R explicitly, restore packages deliberately, and never let package installs happen inside large job arrays.

## Tooling defaults

Slightly opinionated picks for new R projects on the cluster:

- **`rig`** for managing R versions across projects. The cluster module gives you one R; `rig` lets you pin per-project.
- **`renv`** for project libraries. `renv.lock` is the R analogue of `uv.lock` — it is what makes runs reproducible from login node to compute node.
- **`pak`** as the installer behind `renv` for fast parallel installs.
- **`lintr`** + **`styler`** for static checks and formatting; **`testthat`** for tests. Install once at project setup.
- **`optparse`** for CLI scripts (one-file batch entry points like the pattern below).
- **Native pipe `|>`** unless you specifically need magrittr's `%>%` placeholders.

## Style defaults

For the audience that is new to R — these are not stylistic preferences, they are the conventions that will make your code readable to anyone who comes after you (including future-you):

- `snake_case` for variables and functions; reserve `CamelCase` for S4/R6 classes.
- `<-` for assignment, not `=`. (Reserve `=` for named function arguments.)
- All `library()` calls at the top of the script.
- File paths via `here::here()` or explicit project paths; do not hardcode laptop paths.
- Always set `set.seed()` for stochastic work.
- Always use named arguments where the meaning is not obvious: `mean(x, na.rm = TRUE)`, not `mean(x, T)`.
- Smoke test on a small slice of real data before submitting a full cluster job. This is genuinely the fastest debugging loop you have.

## Load R

```bash
module spider r
module load r
R --version
```

Do this in job scripts too. Do not assume `R` is in the default PATH.

## Use renv carefully

In the project directory, on the login node:

```r
install.packages("renv")
renv::init()
renv::install(c("data.table", "arrow", "fixest"))
renv::snapshot()
```

Commit:

```text
renv.lock
.Rprofile
```

Do not commit `renv/library/`.

## Shared library path

For shared projects, set a project library path in `.Rprofile`:

```r
Sys.setenv(RENV_PATHS_LIBRARY = "/gpfs/project/myproject/environments/renv/library")
```

Run `renv::restore()` once during setup, not inside hundreds of jobs. Mutating the project library inside a Slurm array is a metadata storm and a reproducibility hazard.

## Data manipulation defaults

- **tidyverse (`dplyr`, `tidyr`, `readr`)** is the right default for ordinary research code — readable, well-documented, easy to share.
- **`data.table`** (or `dtplyr` for dplyr syntax with data.table speed) when you have actually benchmarked memory or runtime as the bottleneck, typically on data >1GB or in a hot inner loop. Switch because of measured pain, not as an identity marker.
- **`arrow` + Parquet** for anything you store on `/gpfs/project/...` and reuse. One Parquet beats many CSVs for both speed and GPFS metadata health.

## Safe R Slurm template

```bash
#!/bin/bash
#SBATCH --job-name=r-job
#SBATCH --partition=default_queue
#SBATCH --time=01:00:00
#SBATCH --cpus-per-task=4
#SBATCH --mem=16G
#SBATCH --output=logs/%x_%j.out

set -euo pipefail

module purge
module load r

export OMP_NUM_THREADS=${SLURM_CPUS_PER_TASK:-1}
export MKL_NUM_THREADS=${SLURM_CPUS_PER_TASK:-1}
export OPENBLAS_NUM_THREADS=${SLURM_CPUS_PER_TASK:-1}

cd /gpfs/project/myproject/code
srun Rscript src/main.R
```

Use `srun Rscript` (not bare `Rscript`) for the same reason as Python: signals from `--signal=USR1@N` reach the job's main process, not just the batch shell.

## CLI script pattern

Use an explicit script entry point so batch jobs can pass inputs and outputs.

```r
#!/usr/bin/env Rscript
suppressPackageStartupMessages({
  library(optparse)
  library(arrow)
})

option_list <- list(
  make_option(c("-i", "--input"), type = "character"),
  make_option(c("-o", "--output"), type = "character")
)
opt <- parse_args(OptionParser(option_list = option_list))

if (is.null(opt$input) || is.null(opt$output)) {
  stop("--input and --output are required", call. = FALSE)
}
```

Prefer clear errors and command-line arguments over editing paths inside scripts.

## Read Slurm settings in R

```r
slurm_cpus <- Sys.getenv("SLURM_CPUS_PER_TASK", "")
n_cpus <- if (nzchar(slurm_cpus)) as.integer(slurm_cpus) else parallel::detectCores()
job_id <- Sys.getenv("SLURM_JOB_ID", "local")
message("job_id=", job_id, " n_cpus=", n_cpus)

data.table::setDTthreads(n_cpus)
```

## Resumable output

```r
library(arrow)

output <- "/gpfs/project/myproject/output/task_001.parquet"
if (file.exists(output)) {
  message("already done")
  quit(save = "no", status = 0)
}

tmp <- paste0(output, ".tmp")
write_parquet(results, tmp)
file.rename(tmp, output)
```

## Avoid

```bash
Rscript -e 'renv::restore()'   # inside every job: bad
```

Install/restore once, then run many jobs.

## Checklist

- [ ] Job script loads R module explicitly.
- [ ] `renv.lock` is committed.
- [ ] `renv::restore()` is run once at setup, not inside job arrays.
- [ ] Batch scripts accept input/output arguments instead of hardcoded paths.
- [ ] BLAS/OpenMP thread variables are set with `${SLURM_CPUS_PER_TASK:-1}`.
- [ ] `data.table::setDTthreads(n_cpus)` matches allocated CPUs.
- [ ] Long jobs use `srun Rscript ...`.
- [ ] Outputs are resumable and atomically written.

## Further reading

- [renv documentation](https://rstudio.github.io/renv/) — `init`, `snapshot`, `restore`, project libraries.
- [rig](https://github.com/r-lib/rig) — R version manager.
- [pak](https://pak.r-lib.org/) — fast parallel package installer.
- [tidyverse style guide](https://style.tidyverse.org/) — the conventions named above, in depth.
- [data.table reference](https://rdatatable.gitlab.io/data.table/) — `setDTthreads`, `fread`, `:=`, joins.
- [dtplyr](https://dtplyr.tidyverse.org/) — dplyr syntax over data.table.
- [Apache Arrow for R](https://arrow.apache.org/docs/r/) — `open_dataset`, Parquet I/O, dplyr verbs.
- [Lmod user guide](https://lmod.readthedocs.io/en/latest/) — `module load r`, `module purge`.
