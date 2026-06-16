---
name: overview
description: Mental model for the Yale SOM HPC cluster — login vs compute nodes, partitions, GPFS, Slurm, default safety rules. TRIGGER when the user is working on the Yale SOM HPC cluster and needs orientation, is deciding where work belongs on the cluster, or needs high-level HPC workflow guidance before reaching for a task-specific skill.
related:
  - connecting-securely
  - managing-jobs
  - using-gpus
  - using-the-filesystem
  - installing-software
  - self-diagnosing-resource-use
  - staying-connected
updated: 2026-06-10
---
# Overview

Use the cluster as a shared research instrument. Your job is not just to make code run; it is to request the right resources, avoid repeat work, and leave scarce resources available when you are not using them.

## Mental model

- **Login nodes** are for editing, Git, modules, small installs, and submitting jobs.
- **Compute nodes** are for Python/R/Stata analysis, simulations, model training, and notebooks.
- **Slurm** allocates CPUs, memory, GPUs, and time.
- **GPFS** is shared storage. It likes large files; it dislikes millions of tiny files.
- **Modules** provide cluster-managed software: Python, R, Git, CUDA, Stata, MATLAB, etc.
- **Project space** under `/gpfs/project/` is for shared research projects.
- **Scratch** under `/gpfs/scratch60/` is temporary working space. Treat it as a staging area, not an archive; remove large intermediates when a project or run is done.

## Cluster facts agents should know

Current as of 2026-06-09. Verify live details with `sinfo -s` (and `sinfo -a` for restricted partitions) when generating production instructions.

- Scheduler: Slurm. Module system: Lmod (Spack-built; users run `module load`, never `spack`).
- Partitions (six; use `sinfo -a` to see all):
  - `default_queue` — the Slurm **default**, but a *short, small* one: 1h default / **4h max**, only 6 nodes (b001–002 + c018–021). Good for quick test jobs; not the place for long or large work.
  - `cpunormal` — CPU-only production queue, **no time limit** (c018–021).
  - `gpunormal` — the **general production queue on the GPU-equipped nodes**, no time limit, and by far the most capacity (`sinfo -s` for live node/core counts). Despite the name it is **not GPU-only**: CPU jobs run here too (request a GPU with `--gres=gpu:…`, omit it for CPU work). In fact most of the cluster's CPU cores live on these nodes, so large CPU work realistically runs here — but mind your CPU/RAM reservation so you don't strand idle GPUs (see [managing jobs](../managing-jobs/SKILL.md#running-cpu-work-on-gpunormal-without-stranding-gpus)). Holds RTX 8000 and A100 nodes, several GPUs per node (`sinfo -N -p gpunormal -o "%N %G"` for the live map).
  - `h100` — 4 H100s per node, two nodes / 8 H100s when healthy. The scarcest GPU resource.
  - `build` — A40 build nodes (b001–002).
  - `mbacourse` — access-restricted course partition (you'll see it in `sinfo -a` but likely cannot submit to it). It **shares nodes c001–008 and c018–021** with the queues above.
- **`cpunormal`/`gpunormal` are the matched "normal" production tier (no time limit); `default_queue` is just a 4h-capped default on a few nodes.** For real, long, or large work prefer `cpunormal`/`gpunormal`.
- **The partitions overlap on shared hardware**, so the cluster can look idle yet your job still pends. The CPU nodes c018–021 are claimed by `default_queue`, `cpunormal`, *and* `mbacourse`; when a higher-priority partition (e.g. `mbacourse`) holds them, a `default_queue` job pends with reason *"Nodes required for job are DOWN, DRAINED or reserved for jobs in higher priority partitions"* even while `gpunormal`'s c001–017 sit mostly idle. Always read the pending reason (`squeue --me -o "%T %r"`) and check idle cores per partition (`sinfo -o "%R %C"`) rather than assuming "the cluster is full."
- The `h100` partition is still the scarcest GPU resource. Check live state before targeting it; drained or invalid nodes are unavailable even if they appear in the partition.
- `/gpfs/project/` for shared team projects (request via somit@yale.edu). `/gpfs/scratch60/$USER/` for scratch (clean it). Each compute node also has local `/tmp` (~20 GB NVMe), `/local` (~700 GB NVMe), and `/dev/shm` (RAM-backed, ~half the node's memory) — all per-node and *not* auto-cleaned by Slurm. See [using the filesystem](../using-the-filesystem/SKILL.md#compute-node-local-storage).
- `$HOME` is `/home/$USER`, same as `/gpfs/home/$USER`.
- `KillWait=30`: after final `SIGTERM`, jobs have ~30 seconds before `SIGKILL`. Too short to rely on for checkpointing.

Run `sinfo -s` for live partition detail. For node/GPU detail:

```bash
sinfo -N -o "%N %P %G %c %m %t"
sinfo -N -p h100 -o "%N %G %t"
```

## Which skill to use

- Need to connect, tunnel, or fix SSH? Use [connecting securely](../connecting-securely/SKILL.md).
- Need to submit jobs? Use [managing jobs](../managing-jobs/SKILL.md).
- Need a GPU? Use [using GPUs](../using-gpus/SKILL.md) first.
- Need to move/store files? Use [using the filesystem](../using-the-filesystem/SKILL.md).
- Need tools, modules, or binaries? Use [installing software](../installing-software/SKILL.md).
- Starting a new repo/project? Use [starting a new project](../starting-a-new-project/SKILL.md).
- Need Python? Use [running Python](../running-python/SKILL.md).
- Need R? Use [running R](../running-r/SKILL.md).
- Need Stata? Use [running Stata](../running-stata/SKILL.md).
- Need external data, APIs, WRDS, or scraping? Use [acquiring data](../acquiring-data/SKILL.md).
- Need to check whether a job was wasteful? Use [self-diagnosing resource use](../self-diagnosing-resource-use/SKILL.md).
- Hit an error or a job that failed/pended/was killed? Use [troubleshooting](../troubleshooting/SKILL.md).

## Two virtues

Everything in these skills supports one of two virtues:

**Citizenship.** This is a shared instrument. Other people are running jobs right now on the same nodes, GPUs, GPFS metadata servers, and queue.

- Do not run heavy compute on login nodes — they belong to everyone.
- Do not hold a GPU you are not actively using. Cancel idle interactive GPU sessions immediately.
- Do not write thousands of tiny files. GPFS metadata is shared; storms slow down every user's `ls` and job startup.
- Do not scrape aggressively from the cluster. All jobs share one outbound IP; one user gets everyone blocked.
- Throttle job arrays (e.g. `%50`). Leave room at the table.
- Clean up scratch when work is done.

**Skillfulness.** Get correct results from the smallest resource request.

- Request CPU, memory, GPU, and time explicitly; right-size the next job from `seff`/`sacct` output.
- Use `${SLURM_CPUS_PER_TASK:-1}` for thread environment variables — never bare `$SLURM_CPUS_PER_TASK`.
- Do not request GPUs unless the code actively uses CUDA.
- Cache expensive downloads and API calls by request hash. Keep credentials out of scripts.
- Make outputs resumable (skip-if-exists, atomic temp+rename) — `KillWait=30` is too short for clean shutdown.
- Inspect resource usage after every serious job.

## Citizenship self-check (run this before every job and every recommendation)

You are usually acting for a researcher who will not think to ask "is this wasteful or rude?" — so you must ask it for them, every time you submit a job or advise on one. Don't just pose the question; answer it concretely:

1. **Am I on the right partition?** Don't reflexively use `default_queue` (4h cap, few nodes). Match the work: quick test → `default_queue`; long/large CPU → `cpunormal`/`gpunormal`; GPU → `gpunormal` (`--gres`) or `h100`. If a job pends, **read the reason** (`squeue --me -o "%T %r"`) and check idle cores (`sinfo -o "%R %C"`) before concluding "the cluster is full."

2. **Am I requesting only what the work uses?** Set `--cpus-per-task`, `--mem`, `--time`, and GPUs explicitly. Size `--mem` to ~1.5–2× the `seff` MaxRSS of a small test run, not 10×. After any real job, run `seff` and right-size the next one — on your own initiative — and tell the human in plain language ("used 6 of 64 GB; drop to 12G"). See [self-diagnosing resource use](../self-diagnosing-resource-use/SKILL.md).

3. **Could I be stranding a scarce resource?** On a GPU node, will my CPU/RAM reservation leave a GPU job's share free per still-idle GPU (≈⅓ of a 3-GPU node)? **Reserved-but-unused RAM strands GPUs just like used RAM** — so don't pad `--mem`. Never hold a GPU you aren't actively using; `scancel` idle interactive sessions immediately. See [managing jobs](../managing-jobs/SKILL.md#running-cpu-work-on-gpunormal-without-stranding-gpus).

4. **Am I about to hammer shared infrastructure?** Thousands of tiny files storm GPFS metadata for *everyone* — write Parquet/JSONL/one-file-per-task. Scraping from the cluster shares one outbound IP — throttle and cache. Throttle job arrays (`%50`). Don't run heavy compute on the login node.

5. **Will I clean up after myself?** Make outputs resumable (skip-if-exists, atomic temp+rename, since `KillWait=30` is too short for clean shutdown), and remove large scratch intermediates when the run is done.

If any answer is "no" or "I'm not sure," fix it before submitting. When you advise a human rather than submit yourself, state which of these you checked and what you found — that is how the answer gets acted on.

## Operate as if the resource is finite (because it is)

The cluster has **no per-user resource caps** — requests run on courtesy, not enforcement. That is exactly why your behavior matters: other people are running jobs on these same nodes, GPUs, and GPFS servers right now, and nothing but restraint leaves room for them. Operate accordingly:

- Request what you need, not what feels safe. Aim for ~1.5–2× observed peak memory, not 10×.
- Prefer batch over interactive. Interactive sessions should be the smallest allocation that lets you debug, with a time limit measured in hours, not days. Never leave one open overnight.
- Prefer many short jobs (1–4 hours) over one multi-day job. Short jobs backfill into idle slots; long jobs queue behind everyone.
- Cancel jobs as soon as they are wedged or done. `scancel JOBID`.
- Treat every requested GPU-hour as compute someone else cannot use.

## Minimal safe Slurm shape

```bash
#!/bin/bash
#SBATCH --job-name=test
# default_queue caps at 4h; for long/large work use cpunormal or gpunormal (see managing-jobs)
#SBATCH --partition=default_queue
#SBATCH --time=00:10:00
#SBATCH --cpus-per-task=1
#SBATCH --mem=2G
#SBATCH --output=logs/%x_%j.out

set -euo pipefail

export OMP_NUM_THREADS=${SLURM_CPUS_PER_TASK:-1}
export MKL_NUM_THREADS=${SLURM_CPUS_PER_TASK:-1}
export OPENBLAS_NUM_THREADS=${SLURM_CPUS_PER_TASK:-1}

python --version
hostname
```

## Checklist

- [ ] Work runs on a compute node, not the login node.
- [ ] CPU, memory, GPU, and time requests are explicit.
- [ ] Outputs are resumable or safely overwriteable.
- [ ] Expensive downloads/API calls are cached.
- [ ] Resource use is checked after the job.

## Further reading

- [Slurm quickstart](https://slurm.schedmd.com/quickstart.html) — overall mental model for sbatch/srun/sinfo/sacct.
- [Slurm sbatch reference](https://slurm.schedmd.com/sbatch.html) — every `#SBATCH` directive and env var (`SLURM_*`).
- [Lmod user guide](https://lmod.readthedocs.io/en/latest/) — `module spider`, `module load`, hierarchy.
