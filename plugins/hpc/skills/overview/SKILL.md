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

- Scheduler: Slurm.
- Module system: Lmod. Modules are Spack-built, but users only run `module load`, never `spack` directly.
- Main partitions: `default_queue`, `cpunormal`, `gpunormal`, `h100`, `build`.
- `default_queue` has a 1-hour default and a 4-hour max time limit. It spans `b[001-002]` and `c[018-021]`, so a job can land on a build node with an A40 GPU or on a CPU-only node.
- `cpunormal` is for CPU jobs (`c[018-021]`).
- `gpunormal` is for RTX 8000/A100 GPU jobs. A100 nodes hold 3 GPUs each.
- `h100` is the single H100 node `c022` with 4 H100 GPUs.
- `build` (`b[001-002]`) has 1 A40 GPU per node.
- CPU nodes: `c018`–`c021`.
- GPU nodes: `c001`–`c017`, plus `c022` for H100, plus `b001`–`b002` (A40).
- Build nodes: `b001`–`b002`.
- Compute node hostnames resolve internally as `c018.cm.cluster`, `b001.cm.cluster`, etc.
- GPFS home is mounted under `/gpfs/home`. `$HOME` is `/home/$USER`, which resolves to the same place as `/gpfs/home/$USER`.
- `/gpfs/project/` is the right place for shared team projects. Ask somit@yale.edu for a shared directory.
- `/gpfs/scratch60/` is shared scratch and must be cleaned.
- Compute node `/tmp` is local, ephemeral, and small; use it for temporary I/O-heavy work.
- `KillWait=30`: after Slurm sends final `SIGTERM`, jobs have only about 30 seconds before `SIGKILL`.

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

## Default safety rules

1. Do not run heavy compute on login nodes.
2. Do not request GPUs unless code actively uses them.
3. Do not write thousands of tiny files when one Parquet/HDF5/zip file will do.
4. Do not put credentials or API keys in scripts.
5. Do not repeat paid API calls; cache by request hash.
6. Do not rely on final `SIGTERM` for checkpointing; 30 seconds is too short.
7. Do not use bare `$SLURM_CPUS_PER_TASK` in shell examples; use `${SLURM_CPUS_PER_TASK:-1}`.
8. Always inspect resource usage after a serious job.

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
