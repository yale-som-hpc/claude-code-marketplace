---
name: managing-jobs
description: Submit, monitor, cancel, array, and chain Slurm jobs on the Yale SOM HPC cluster. TRIGGER when writing sbatch scripts for the Yale SOM HPC cluster, choosing cluster partitions/resources, using job arrays or dependencies on the cluster, or running sacct/squeue/scancel against cluster jobs.
related:
  - overview
  - self-diagnosing-resource-use
  - running-python
  - running-r
  - running-stata
  - using-gpus
updated: 2026-04-28
---
# Managing Jobs

Rule: test small, request explicitly, monitor results, then scale.

## Basic commands

```bash
sbatch job.sh                    # submit batch job
squeue -u $USER                  # current jobs
sacct -j 12345                   # completed job accounting
scancel 12345                    # cancel your job
scontrol show job 12345          # detailed job state
sinfo -s                         # partition summary
```

## Safe Slurm template

```bash
#!/bin/bash
#SBATCH --job-name=analysis
#SBATCH --partition=default_queue
#SBATCH --time=00:30:00
#SBATCH --cpus-per-task=4
#SBATCH --mem=16G
#SBATCH --output=logs/%x_%j.out

set -euo pipefail

export OMP_NUM_THREADS=${SLURM_CPUS_PER_TASK:-1}
export MKL_NUM_THREADS=${SLURM_CPUS_PER_TASK:-1}
export OPENBLAS_NUM_THREADS=${SLURM_CPUS_PER_TASK:-1}

cd /gpfs/project/myproject/code
uv run python src/analysis.py
```

## Interactive session

Use only for debugging, never for long unattended work. Interactive jobs hold resources whether or not you are typing — a forgotten `srun --pty bash` can sit on a GPU all weekend.

```bash
srun --partition=cpunormal --cpus-per-task=2 --mem=8G --time=01:00:00 --pty bash
```

Rules:

- Smallest allocation that lets you debug. Two CPUs and 8 GB is usually enough.
- `--time` measured in hours, not days. Re-request if you need more.
- Exit (`exit` or `Ctrl-D`) the moment you are done. Do not minimize the terminal and walk away.
- Anything that can run unattended belongs in `sbatch`, not `srun --pty`.

## Job arrays

Use arrays for independent tasks. Throttle concurrency with `%N`.

```bash
#!/bin/bash
#SBATCH --job-name=array-example
#SBATCH --partition=default_queue
#SBATCH --array=1-500%50
#SBATCH --time=00:30:00
#SBATCH --cpus-per-task=1
#SBATCH --mem=2G
#SBATCH --output=logs/%x_%A_%a.out

set -euo pipefail

export OMP_NUM_THREADS=${SLURM_CPUS_PER_TASK:-1}
export MKL_NUM_THREADS=${SLURM_CPUS_PER_TASK:-1}
export OPENBLAS_NUM_THREADS=${SLURM_CPUS_PER_TASK:-1}

uv run python src/run_task.py --task-id "${SLURM_ARRAY_TASK_ID}"
```

Do not submit 500 tasks all at once unless you mean to occupy the cluster. Use `%50` or smaller.

Politeness rule: leave room at the table. If your array can run at 50-way concurrency instead of 500-way, choose 50. Other people are using the same nodes, GPUs, GPFS metadata servers, and queue. Throttling your own work usually also makes debugging easier.

## Dependencies

```bash
prep=$(sbatch --parsable slurm/01_prepare.sh)
est=$(sbatch --parsable --dependency=afterok:${prep} slurm/02_estimate.sh)
sbatch --dependency=afterok:${est} slurm/03_tables.sh
```

Use `afterany` for cleanup or restart logic that should run even after failure.

## Time limits

Shorter jobs often schedule faster because Slurm can backfill them into idle slots between bigger jobs. Multi-day jobs queue behind everyone. If work is resumable, prefer 2–4 hour chunks; with skip-if-exists outputs, a killed job picks up where it left off on resubmit.

## Right-size before submitting

Do not pad requests "just in case." Over-requesting blocks scheduling for everyone, and the cluster is moving toward enforced per-user caps. The right-sizing loop:

1. Submit a 10-minute test job with a small input.
2. Run `seff JOBID` after it finishes.
3. Set the real job's `--mem` to ~1.5–2× the test's `MaxRSS`, not 10×.
4. Set `--cpus-per-task` to what your code actually parallelizes over (`SLURM_CPUS_PER_TASK` controls BLAS, multiprocessing, `setDTthreads`, `set processors`).
5. Set `--time` from a sample-data extrapolation, not from "what if it takes a week."

See [self-diagnosing resource use](../self-diagnosing-resource-use/SKILL.md) for the post-job checks that drive this loop.

## Before scaling up

```bash
sbatch slurm/test.sh
squeue -u $USER
sacct -j JOBID --format=JobID,Elapsed,MaxRSS,AllocCPUS,TotalCPU,State
```

## Checklist

- [ ] Job starts with a small test.
- [ ] `--time`, `--mem`, and `--cpus-per-task` are explicit.
- [ ] Thread variables are set with `${SLURM_CPUS_PER_TASK:-1}`.
- [ ] Arrays use a concurrency throttle like `%50`.
- [ ] Output paths include job IDs or task IDs.
- [ ] Resource usage is checked after completion.

## Further reading

- [Slurm sbatch reference](https://slurm.schedmd.com/sbatch.html) — every `#SBATCH` directive, output filename patterns, signal handling.
- [Slurm job arrays](https://slurm.schedmd.com/job_array.html) — `--array` syntax, throttling (`%N`), `SLURM_ARRAY_*` env vars.
- [Slurm squeue](https://slurm.schedmd.com/squeue.html) and [sacct](https://slurm.schedmd.com/sacct.html) — format strings, state codes.
- [Slurm quickstart](https://slurm.schedmd.com/quickstart.html) — sbatch/srun/sacct big picture.
