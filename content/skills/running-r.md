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

## Avoid

```bash
Rscript -e 'renv::restore()'   # inside every job: bad
```

Install/restore once, then run many jobs.

## Checklist

- [ ] Job script loads R module explicitly.
- [ ] `renv.lock` is committed.
- [ ] `renv::restore()` is not inside job arrays.
- [ ] BLAS/OpenMP thread variables are set.
- [ ] `data.table` threads match allocated CPUs.
- [ ] Outputs are resumable and atomically written.
