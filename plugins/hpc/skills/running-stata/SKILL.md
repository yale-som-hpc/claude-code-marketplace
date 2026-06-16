---
name: running-stata
description: Run Stata batch jobs on the Yale SOM HPC cluster with logs, scratch temp, CPU matching, and license courtesy. TRIGGER when writing Stata sbatch scripts for the Yale SOM HPC cluster, choosing Stata/MP cores there, or diagnosing Stata batch jobs/licenses on the cluster.
related:
  - managing-jobs
  - using-the-filesystem
  - accelerating-python
  - self-diagnosing-resource-use
updated: 2026-06-10
---
# Running Stata

Rule: run Stata in batch jobs, write logs, put temp files on scratch, and request only the cores Stata will use.

What you get: the `stata` module is a complete MP install — no package-bootstrap step (unlike R, whose base module ships nothing). `ssc install` packages land in your personal ado directory.

## Slurm template

```bash
#!/bin/bash
#SBATCH --job-name=stata-job
# default_queue caps at 4h; for long/large work use cpunormal or gpunormal (see managing-jobs)
#SBATCH --partition=default_queue
#SBATCH --time=01:00:00
#SBATCH --cpus-per-task=4
#SBATCH --mem=16G
#SBATCH --output=logs/%x_%j.out

set -euo pipefail

module purge
module load stata

# These only affect external BLAS calls; native Stata/MP threading is controlled
# by `set processors` (below), not by these. Harmless to keep for portability.
export OMP_NUM_THREADS=${SLURM_CPUS_PER_TASK:-1}
export OPENBLAS_NUM_THREADS=${SLURM_CPUS_PER_TASK:-1}
# The stata/19 module defaults STATATMP=/gpfs/scratch60; override for per-job isolation:
export STATATMP=/gpfs/scratch60/$USER/stata-tmp/${SLURM_JOB_ID}

mkdir -p "$STATATMP" logs
trap 'rm -rf "$STATATMP"' EXIT

cd /gpfs/project/myproject/code
stata-mp -b do src/main.do
```

Two log gotchas, both verified on the cluster:

- **Read the `.log`, not the `.out`.** Under `stata-mp -b`, Stata sends all output to its own log file; the Slurm `--output` `.out` file ends up **empty**. The `log using` inside your do-file (below) is what you read.
- **`stata-mp -b do src/main.do` also writes `main.log` in the working directory** (named after the do-file), in addition to your `log using`. Either expect/clean up that stray file or name your `log using` differently.
- **Check the log for errors — batch mode can exit 0 on a Stata error.** Grep the `.log` for an `r(###)` return code; `set -e` will not catch a do-file error on its own.

## Do-file preamble

```stata
capture log close _all
local jobid : env SLURM_JOB_ID
if "`jobid'" == "" local jobid local
log using "logs/main_`jobid'.log", replace text

local ncpus : env SLURM_CPUS_PER_TASK
if "`ncpus'" == "" local ncpus 1
set processors `ncpus'

local statatmp : env STATATMP
di "STATATMP=`statatmp'"

set more off
version 19
```

## Memory hygiene

```stata
compress
describe
memory report
```

Use `frames`, `tempfile`, and `preserve`/`restore` deliberately. Drop unneeded variables before merges.

## Job arrays

```stata
local task_id : env SLURM_ARRAY_TASK_ID
if "`task_id'" == "" local task_id 1

di "Running task `task_id'"
```

Each array task should write a separate output file.

## License courtesy

Stata MP licenses are shared. Do not leave interactive Stata sessions idle. Do not request more CPUs than `set processors` uses.

## Checklist

- [ ] Stata runs via `stata-mp -b do`, not interactively on the login node.
- [ ] Logs are written to `logs/`.
- [ ] `STATATMP` points to scratch and is cleaned up.
- [ ] `set processors` matches `SLURM_CPUS_PER_TASK`.
- [ ] Each array task writes separate output.
- [ ] Idle Stata sessions are closed.

## Further reading

- [Stata batch mode (`gswb` PDF)](https://www.stata.com/manuals/gswb.pdf) — `-b do`, log behavior, exit codes.
- [Stata `set processors`](https://www.stata.com/help.cgi?set_processors) — MP core control.
- [Stata `frames` reference](https://www.stata.com/manuals/dframes.pdf) — multiple in-memory datasets.
