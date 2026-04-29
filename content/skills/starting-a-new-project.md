---
title: Starting a New Project
slug: starting-a-new-project
description: Create a research project layout that is reproducible, resumable, and safe for shared HPC use.
related:
  - installing-software
  - managing-jobs
  - running-python
  - running-r
  - using-the-filesystem
  - acquiring-data
updated: 2026-04-28
---

# Starting a New Project

Rule: make the project understandable to a new RA and restartable by a future you.

## Recommended layout

```text
/gpfs/project/myproject/
├── code/                 # Git repo
│   ├── README.md
│   ├── pyproject.toml
│   ├── uv.lock
│   ├── renv.lock
│   ├── src/
│   ├── scripts/
│   └── slurm/
├── data/
│   ├── raw/              # read-only original data
│   └── derived/          # rebuildable intermediates
├── output/               # tables, figures, final outputs
├── logs/                 # Slurm logs
└── cache/                # API/download/model caches
```

## First commands

```bash
mkdir -p /gpfs/project/myproject/{code,data/raw,data/derived,output,logs,cache}
cd /gpfs/project/myproject/code
git init
uv init --app
mkdir -p src scripts slurm
```

## `.gitignore`

```gitignore
.venv/
__pycache__/
*.pyc
.Rproj.user/
renv/library/
logs/
output/
data/raw/
data/derived/
.env
```

Track code, lockfiles, and documentation. Do not track data, credentials, environments, logs, or generated output.

## README minimum

```markdown
# Project Name

## Setup

1. Clone repo into `/gpfs/project/myproject/code`.
2. Run `uv sync` on the login node.
3. If using R, run `Rscript -e 'renv::restore()'` on the login node.
4. Submit `sbatch slurm/test.sh`.

## Pipeline

1. `sbatch slurm/01_prepare_data.sh`
2. `sbatch slurm/02_estimate.sh`
3. Outputs appear in `/gpfs/project/myproject/output/`.

## Data

- `data/raw/`: original data, do not edit.
- `data/derived/`: rebuildable from raw data and code.
```

## Add a thin Justfile

Use `just` to document common commands without hiding what they do.

```just
set dotenv-load
set shell := ["bash", "-euo", "pipefail", "-c"]

default:
    @just --list

setup:
    uv sync

run *ARGS:
    uv run python src/main.py {{ARGS}}

test-job:
    sbatch slurm/test.sh
```

Keep recipes thin. The real logic should live in Python/R/Stata scripts, not in the Justfile.

## First test job

```bash
#!/bin/bash
#SBATCH --job-name=myproject-test
#SBATCH --partition=default_queue
#SBATCH --time=00:10:00
#SBATCH --cpus-per-task=1
#SBATCH --mem=2G
#SBATCH --output=/gpfs/project/myproject/logs/%x_%j.out

set -euo pipefail

export OMP_NUM_THREADS=${SLURM_CPUS_PER_TASK:-1}
export MKL_NUM_THREADS=${SLURM_CPUS_PER_TASK:-1}
export OPENBLAS_NUM_THREADS=${SLURM_CPUS_PER_TASK:-1}

cd /gpfs/project/myproject/code
uv run python -c "import sys, pathlib; print(sys.version); print(pathlib.Path.cwd())"
```

## Make raw data read-only after ingest

After the raw-data ingest is complete and checked, freeze it:

```bash
chmod -R g-w /gpfs/project/myproject/data/raw
```

Do not lock `data/raw/` before collaborators have finished placing the initial files there.

## Checklist

- [ ] Project lives in `/gpfs/project/...`, not one person's home directory.
- [ ] Code is in Git.
- [ ] Raw data is read-only.
- [ ] Environments are reproducible from lockfiles.
- [ ] Logs go to `logs/`.
- [ ] Justfile or README documents the common commands.
- [ ] First Slurm test job passes before any full run.
