---
name: running-stata
description: Run Stata batch jobs with logs, scratch temp space, CPU matching, and license courtesy. TRIGGER when running Stata do-files, choosing Stata/MP cores, writing Stata sbatch scripts, or diagnosing Stata logs/licenses.
related:
  - managing-jobs
  - using-the-filesystem
  - working-with-large-data
  - self-diagnosing-resource-use
updated: 2026-04-28
---
# Running Stata

Rule: run Stata in batch jobs, write logs, put temp files on scratch, and request only the cores Stata will use.

## Slurm template

```bash
#!/bin/bash
#SBATCH --job-name=stata-job
#SBATCH --partition=default_queue
#SBATCH --time=01:00:00
#SBATCH --cpus-per-task=4
#SBATCH --mem=16G
#SBATCH --output=logs/%x_%j.out

set -euo pipefail

export OMP_NUM_THREADS=${SLURM_CPUS_PER_TASK:-1}
export MKL_NUM_THREADS=${SLURM_CPUS_PER_TASK:-1}
export OPENBLAS_NUM_THREADS=${SLURM_CPUS_PER_TASK:-1}
export STATATMP=/gpfs/scratch60/$USER/stata-tmp/${SLURM_JOB_ID}

mkdir -p "$STATATMP" logs
trap 'rm -rf "$STATATMP"' EXIT

cd /gpfs/project/myproject/code
stata-mp -b do src/main.do
```

## Do-file preamble

```stata
capture log close _all
log using "logs/main_${SLURM_JOB_ID}.log", replace text

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
