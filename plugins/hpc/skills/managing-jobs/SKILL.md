---
name: managing-jobs
description: Submit, monitor, cancel, array, and chain Slurm jobs on the Yale SOM HPC cluster. TRIGGER when running, submitting, or scheduling any job or analysis on the cluster (even phrased plainly, e.g. "run my regression on the cluster"), writing sbatch scripts, choosing partitions/resources, or using job arrays, dependencies, or sacct/squeue/scancel.
related:
  - overview
  - self-diagnosing-resource-use
  - running-python
  - running-r
  - running-stata
  - using-gpus
updated: 2026-06-10
---
# Managing Jobs

Rule: test small, request explicitly, monitor results, then scale.

Before you submit a job — or recommend one to a human — ask **"am I being a good citizen?"** and answer it: right partition for the work, requested only what it uses (right-size `--mem`/`--cpus`/`--time`), not stranding idle GPUs with a CPU/RAM reservation, not storming GPFS with tiny files, and cleaning up after. The concrete how-to-answer for each is the [citizenship self-check in overview](../overview/SKILL.md#citizenship-self-check-run-this-before-every-job-and-every-recommendation). When you advise rather than submit, tell the human which checks you ran.

## Basic commands

```bash
sbatch job.sh                    # submit batch job
squeue -u $USER                  # current jobs
sacct -j 12345                   # completed job accounting
scancel 12345                    # cancel your job
scontrol show job 12345          # detailed job state
sinfo -s                         # partition summary
```

Common job states: `R` is running, `PD` is pending, and `CG` is completing. If a job stays pending unusually long, inspect the reason instead of just waiting; the request may not fit available nodes, memory, GPUs, partition limits, or time limits.

```bash
squeue --me --start
scontrol show job JOBID
```

## Choosing a partition

`default_queue` is the Slurm default, but it is a *short, small* queue (1h default / 4h max, only 6 nodes) and those nodes are shared with — and outranked by — the restricted `mbacourse` partition. So a `default_queue` job can pend with *"Nodes required for job are DOWN, DRAINED or reserved for jobs in higher priority partitions"* **even when the cluster is mostly idle**, because the idle cores are on partitions it doesn't target.

Before assuming "the cluster is full," look:

```bash
squeue --me -o "%.10i %.9P %.2t %.10M %r"   # %r = pending reason — read it
sinfo -o "%R %C"                            # idle cores per partition (A/I/O/T)
```

Pick the partition that fits the work:

- **Quick test jobs (≤4h):** `default_queue` is fine.
- **Real / long / large CPU work:** `cpunormal` or `gpunormal` — these are the "normal" production queues with **no time limit**. `gpunormal` is the largest pool and, despite the name, takes CPU-only jobs (just omit `--gres`). When `default_queue`/`cpunormal` show 0 idle cores, a CPU job submitted to `gpunormal` typically starts immediately.
- **GPU work:** `gpunormal` (`--gres=gpu:1`), or `h100` for H100s. See [using GPUs](../using-gpus/SKILL.md).

### Running CPU work on `gpunormal` without stranding GPUs

Most of the cluster's CPU cores live on the GPU nodes (the CPU-only partitions are just a handful of nodes), so substantial CPU work often *has* to run on `gpunormal`. The hazard: **Slurm reserves the CPU and RAM you request whether or not you use them**, and a GPU job needs CPU + RAM *alongside* its GPU. If your CPU-only job doesn't leave a GPU job's worth of headroom per still-idle GPU on that node, those GPUs become unschedulable — scarce hardware sits idle.

This happens for real: a CPU-only job reserving ~900 GB on a 1 TB, 3-GPU node leaves only ~120 GB — room for one GPU job, stranding the other two GPUs, even though `nvidia-smi` shows them idle and the OS shows the RAM physically free. Reserved-but-unused is just as blocking as used.

So when you must run CPU-only work on `gpunormal`:

- **Right-size `--mem` and `--cpus-per-task` from `seff` — never pad "just in case."** This is the single biggest cause of accidental stranding.
- **Leave a GPU's share free.** A GPU job here typically needs roughly **8 CPUs and ~120 GB per GPU** (check live with `squeue -p gpunormal -t R -O NumCPUs,MinMemory,tres-per-node`). On 3-GPU nodes, reserve CPU-only work in slices that leave about **24 CPUs and 360 GB** free if the GPUs are still idle.
- **For big CPU arrays, do not submit thousands of independent 1-core array jobs directly to `gpunormal`.** A global array throttle (`%200`) does not protect per-node headroom; Slurm can still pack the jobs onto GPU nodes until their CPUs/RAM are full. Prefer one **node-slice worker** allocation: one modest CPU/RAM slice per node, then run many serial tasks inside that slice.
- **Keep big CPU jobs off the scarcest GPU nodes** — prefer `cpunormal`/`default_queue`, or the RTX 8000 / 40 GB A100 nodes, over the 80 GB A100 and H100 nodes.
- **Check what you'd be sitting next to:** `sinfo -N -p gpunormal -O NodeHost,CPUsState,FreeMem,Gres,GresUsed` shows nodes with idle GPUs (`GresUsed` < `Gres`) whose CPU/RAM you'd be tying up.

Good patterns for CPU-only batch work on `gpunormal`:

**Pattern A: node-slice worker for many single-core commands** (best for CmdStan chains, simulations, bootstrap reps, etc.). First make a manifest with one command per line:

```bash
# commands.txt: one independent serial task per line
/gpfs/home/me/cmdstan/model sample data file=data/a.json output file=results/a.csv
/gpfs/home/me/cmdstan/model sample data file=data/b.json output file=results/b.csv
```

Submit a few bounded slices, not thousands of tiny Slurm jobs:

```bash
#!/bin/bash
#SBATCH --job-name=cpu-slice
#SBATCH --partition=gpunormal
#SBATCH --nodes=4                 # start small; scale after checking queue impact
#SBATCH --ntasks=4                # match --nodes: one worker task per node
#SBATCH --ntasks-per-node=1        # one worker per node
#SBATCH --cpus-per-task=32         # run at most 32 serial commands per node
#SBATCH --mem=32G                  # right-size; e.g. 1G per command, not 4G if it uses 80M
#SBATCH --time=04:00:00
#SBATCH --output=logs/%x_%j_%t.out

set -euo pipefail
export OMP_NUM_THREADS=1
export MKL_NUM_THREADS=1
export OPENBLAS_NUM_THREADS=1

srun --ntasks="$SLURM_NTASKS" --ntasks-per-node=1 bash worker.sh commands.txt
```

`worker.sh` shards the manifest across workers and uses local parallelism inside each slice:

```bash
#!/bin/bash
set -euo pipefail
cmdfile=$1

awk -v r="$SLURM_PROCID" -v n="$SLURM_NTASKS" '((NR - 1) % n) == r' "$cmdfile" |
  parallel --line-buffer --halt soon,fail=1 -j "$SLURM_CPUS_PER_TASK" --joblog "logs/${SLURM_JOB_ID}_${SLURM_PROCID}.joblog"
```

Why this is good: the example runs 128 serial tasks at once, but reserves only 32 CPUs / 32 GB per node. On a 128-core, 3-GPU node it leaves about 96 CPUs and most memory available, so GPU jobs can still land.

**Pattern B: if you keep a Slurm array, make it small and low-memory.** This is simpler but less protective because Slurm can still pack tasks unevenly:

```bash
#SBATCH --partition=cpunormal,gpunormal
#SBATCH --array=1-1000%100         # cap total concurrency; lower this if GPU jobs are pending
#SBATCH --cpus-per-task=1
#SBATCH --mem=1G                   # set from seff/MaxRSS; do not pad to 4G if it uses <1G
#SBATCH --time=04:00:00
```

When scarce GPU jobs are pending, either lower the throttle further or temporarily avoid the A100 nodes:

```bash
#SBATCH --exclude=c009-c017        # avoid A100 nodes for CPU-only work when needed
```

## Safe Slurm template

```bash
#!/bin/bash
#SBATCH --job-name=analysis
# default_queue caps at 4h and is small; for long/large work use cpunormal or gpunormal (see "Choosing a partition")
#SBATCH --partition=default_queue
#SBATCH --time=00:30:00
#SBATCH --cpus-per-task=4
#SBATCH --mem=16G
#SBATCH --output=logs/%x_%j.out

set -euo pipefail

export OMP_NUM_THREADS=${SLURM_CPUS_PER_TASK:-1}
export MKL_NUM_THREADS=${SLURM_CPUS_PER_TASK:-1}
export OPENBLAS_NUM_THREADS=${SLURM_CPUS_PER_TASK:-1}
export PYTHONUNBUFFERED=1

cd /gpfs/project/myproject/code
# Environment was created during setup with: uv sync --frozen
srun .venv/bin/python src/analysis.py
```

Use `srun .venv/bin/python ...` (not `uv run python ...` and not bare `python` under bash) for any job that may run long enough to hit `--signal=USR1@N`. See [running Python](../running-python/SKILL.md) and [parallel Python](../parallel-python/SKILL.md) for the empirical reasons. For R, the equivalent is `srun Rscript src/main.R`.

A note on `SLURM_*` environment variables: these are only set inside `sbatch`/`srun` jobs. The `${SLURM_CPUS_PER_TASK:-1}` form means "use Slurm's value when set, otherwise fall back to 1," which lets you run the same script on your laptop (where it falls back to 1 thread) and on the cluster (where Slurm fills it in). Same idea for `SLURM_JOB_ID`, `SLURM_ARRAY_TASK_ID`, etc.

## Bad-interpreter / `$'\r'` errors from sbatch

If `sbatch job.sh` fails with `bad interpreter`, `: not found`, or `$'\r'` in the error, you have CRLF line endings (typical when editing on Windows or when a download mangled the file). Fix in place:

```bash
dos2unix job.sh
# or, without dos2unix installed:
sed -i 's/\r$//' job.sh
```

Configure your editor to write LF for `.sh` files going forward.

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
# default_queue caps at 4h and is small; for long/large work use cpunormal or gpunormal (see "Choosing a partition")
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
export PYTHONUNBUFFERED=1

srun .venv/bin/python src/run_task.py --task-id "${SLURM_ARRAY_TASK_ID}"
```

Do not submit 500 tasks all at once unless you mean to occupy the cluster. Use `%50` or smaller.

Politeness rule: leave room at the table. If your array can run at 50-way concurrency instead of 500-way, choose 50. Other people are using the same nodes, GPUs, GPFS metadata servers, and queue. Throttling your own work usually also makes debugging easier.

### Use a manifest, not glob order

Do not have task `i` index into `sorted(glob("data/*.parquet"))[i]` — `ls`/glob order is not stable across machines or filesystem states, and adding a new file mid-array silently shifts every subsequent task to the wrong input. Build a manifest once at setup and have the array index into it:

```bash
# At setup (login node), once:
ls /gpfs/project/myproject/data/raw/*.parquet > slurm/manifest.txt
n=$(wc -l < slurm/manifest.txt)
echo "manifest has $n tasks; submit with --array=1-$n%50"
```

```bash
# In the array job script:
input=$(sed -n "${SLURM_ARRAY_TASK_ID}p" slurm/manifest.txt)
srun .venv/bin/python src/run_task.py --input "$input"
```

Commit `slurm/manifest.txt` so reruns and collaborators see the same task → input mapping.

### Per-job temp directory

For high-I/O work, stage onto compute-node `/tmp` and copy results back. The recipe lives in [using the filesystem](../using-the-filesystem/SKILL.md). The short version: `mktemp -d "${TMPDIR:-/tmp}/job_${SLURM_JOB_ID:-local}.XXXXXX"`, set `TMPDIR`, `trap` to clean.

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

Do not pad requests "just in case." Over-requesting blocks scheduling for everyone else on a shared cluster (there are no per-user caps — it runs on courtesy). The right-sizing loop:

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

## Report back without being asked

When a real job finishes, do not stop at "it ran." Proactively run `seff JOBID` and tell the user, in plain language, whether the job used what it asked for — e.g. "it used 6 GB of the 64 GB requested and 1 of 4 CPUs, so next time request `--mem=12G --cpus-per-task=1`." Most researchers will not think to ask; surfacing waste is part of the job, not an extra. See [self-diagnosing resource use](../self-diagnosing-resource-use/SKILL.md).

## Checklist

- [ ] Job starts with a small test.
- [ ] `--time`, `--mem`, and `--cpus-per-task` are explicit.
- [ ] Thread variables are set with `${SLURM_CPUS_PER_TASK:-1}`; `PYTHONUNBUFFERED=1` for Python jobs.
- [ ] Long jobs launch with `srun .venv/bin/python ...` (or `srun Rscript ...`), not `uv run python ...`.
- [ ] Arrays use a concurrency throttle like `%50` and index into a stable manifest, not glob order.
- [ ] Output paths include job IDs or task IDs.
- [ ] Job script is LF-terminated (no CRLF) so sbatch does not fail with `$'\r'`.
- [ ] Resource usage is checked after completion.

## Further reading

- [Slurm sbatch reference](https://slurm.schedmd.com/sbatch.html) — every `#SBATCH` directive, output filename patterns, signal handling.
- [Slurm job arrays](https://slurm.schedmd.com/job_array.html) — `--array` syntax, throttling (`%N`), `SLURM_ARRAY_*` env vars.
- [Slurm squeue](https://slurm.schedmd.com/squeue.html) and [sacct](https://slurm.schedmd.com/sacct.html) — format strings, state codes.
- [Slurm quickstart](https://slurm.schedmd.com/quickstart.html) — sbatch/srun/sacct big picture.
