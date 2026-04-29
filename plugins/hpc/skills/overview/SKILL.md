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
updated: 2026-04-28
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

Current as of 2026-04-28. Verify live details with `sinfo -s` when generating production instructions.

- Scheduler: Slurm. Module system: Lmod (Spack-built; users run `module load`, never `spack`).
- Main partitions: `default_queue` (4h max, mixed CPU/A40), `cpunormal`, `gpunormal` (RTX 8000/A100, 3 GPUs per A100 node), `h100` (one node, 4 H100s — scarcest resource on the cluster), `build`.
- `/gpfs/project/` for shared team projects (request via somit@yale.edu). `/gpfs/scratch60/$USER/` for scratch (clean it). Compute node `/tmp` is local and ephemeral.
- `$HOME` is `/home/$USER`, same as `/gpfs/home/$USER`.
- `KillWait=30`: after final `SIGTERM`, jobs have ~30 seconds before `SIGKILL`. Too short to rely on for checkpointing.

Run `sinfo -s` for live node-level detail.

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

## Two pillars

Everything in these skills supports one of two goals:

**Be polite.** This is a shared instrument. Other people are running jobs right now on the same nodes, GPUs, GPFS metadata servers, and queue.

- Do not run heavy compute on login nodes — they belong to everyone.
- Do not hold a GPU you are not actively using. Cancel idle interactive GPU sessions immediately.
- Do not write thousands of tiny files. GPFS metadata is shared; storms slow down every user's `ls` and job startup.
- Do not aggressive-scrape from the cluster. All jobs share one outbound IP; one user gets everyone blocked.
- Throttle job arrays (e.g. `%50`). Leave room at the table.
- Clean up scratch when work is done.

**Be skillful.** Get correct results from the smallest resource request.

- Request CPU, memory, GPU, and time explicitly; right-size the next job from `seff`/`sacct` output.
- Use `${SLURM_CPUS_PER_TASK:-1}` for thread environment variables — never bare `$SLURM_CPUS_PER_TASK`.
- Do not request GPUs unless the code actively uses CUDA.
- Cache expensive downloads and API calls by request hash. Keep credentials out of scripts.
- Make outputs resumable (skip-if-exists, atomic temp+rename) — `KillWait=30` is too short for clean shutdown.
- Inspect resource usage after every serious job.

## Treat caps as if they exist

Today the cluster has light enforcement of per-user resource caps, but that is not a design choice — it is a courtesy. Other clusters (Grace, most NSF/DOE systems) have hard partition caps, monthly compute budgets, and tight interactive limits. SOM HPC is moving in that direction.

Operate as if the rules already apply:

- Request what you need, not what feels safe. Aim for ~1.5–2× observed peak memory, not 10×.
- Prefer batch over interactive. Interactive sessions should be the smallest allocation that lets you debug, with a time limit measured in hours, not days. Never leave one open overnight.
- Prefer many short jobs (1–4 hours) over one multi-day job. Short jobs backfill into idle slots; long jobs queue behind everyone.
- Cancel jobs as soon as they are wedged or done. `scancel JOBID`.
- Treat every requested GPU-hour as compute someone else cannot use.

## Minimal safe Slurm shape

```bash
#!/bin/bash
#SBATCH --job-name=test
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
