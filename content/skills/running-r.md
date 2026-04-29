---
title: Running R
slug: running-r
description: Run R with modules, renv, batch scripts, thread control, and safe library paths.
related:
  - installing-software
  - managing-jobs
  - using-the-filesystem
  - working-with-large-data
updated: 2026-04-28
---

# Running R

Rule: load R explicitly, restore packages deliberately, and never let package installs happen inside large job arrays.

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

Run `renv::restore()` once during setup, not inside hundreds of jobs.

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
Rscript src/main.R
```

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
n_cpus <- as.integer(Sys.getenv("SLURM_CPUS_PER_TASK", "1"))
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

## Style defaults

- Use `snake_case` for variables and functions.
- Use `<-` for assignment.
- Put `library()` calls at the top.
- Use `here::here()` or explicit project paths; do not hardcode laptop paths.
- Use `set.seed()` for stochastic work.
- Prefer smoke tests on small real data before full cluster runs.

## Avoid

```bash
Rscript -e 'renv::restore()'   # inside every job: bad
```

Install/restore once, then run many jobs.

## Checklist

- [ ] Job script loads R module explicitly.
- [ ] `renv.lock` is committed.
- [ ] `renv::restore()` is not inside job arrays.
- [ ] Batch scripts accept input/output arguments instead of hardcoded paths.
- [ ] BLAS/OpenMP thread variables are set.
- [ ] `data.table` threads match allocated CPUs.
- [ ] Outputs are resumable and atomically written.
