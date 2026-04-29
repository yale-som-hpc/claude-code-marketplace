---
name: running-python
description: Run Python on the Yale SOM HPC cluster with uv, Slurm, thread control, logging, and resumable outputs. TRIGGER when writing Python sbatch scripts for the Yale SOM HPC cluster, creating uv environments under /gpfs, or debugging Python Slurm jobs on the cluster.
related:
  - installing-software
  - managing-jobs
  - parallel-python
  - accelerating-python
  - working-with-large-data
  - using-gpus
  - acquiring-data
updated: 2026-04-28
---
# Running Python

Rule: use a project environment, control threads, log clearly, and make outputs resumable.

## Project setup with uv

```bash
cd /gpfs/project/myproject/code
uv init --app
uv add polars pyarrow duckdb
uv sync
```

Commit these:

```text
pyproject.toml
uv.lock
```

Do not commit `.venv/`, put it in `.gitignore`

## Safe Python Slurm template

```bash
#!/bin/bash
#SBATCH --job-name=python-job
#SBATCH --partition=default_queue
#SBATCH --time=01:00:00
#SBATCH --cpus-per-task=4
#SBATCH --mem=16G
#SBATCH --output=logs/%x_%j.out

set -euo pipefail

export OMP_NUM_THREADS=${SLURM_CPUS_PER_TASK:-1}
export MKL_NUM_THREADS=${SLURM_CPUS_PER_TASK:-1}
export OPENBLAS_NUM_THREADS=${SLURM_CPUS_PER_TASK:-1}
export NUMEXPR_NUM_THREADS=${SLURM_CPUS_PER_TASK:-1}

cd /gpfs/project/myproject/code
uv run python src/main.py
```

## Read Slurm settings safely

```python
import os

n_cpus = int(os.environ.get("SLURM_CPUS_PER_TASK", "1"))
job_id = os.environ.get("SLURM_JOB_ID", "local")
```

## Logging

```python
import logging
import os

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s %(levelname)s %(message)s",
)

logging.info("job_id=%s", os.environ.get("SLURM_JOB_ID", "local"))
```

Use logs to know what happened without opening notebooks.

## Multiprocessing

For nontrivial multiprocessing, producer-consumer queues, process pools, or job-stealing-style work, use [parallel Python](../parallel-python/SKILL.md).

Minimal rule: match worker count to allocated CPUs.

```python
from concurrent.futures import ProcessPoolExecutor
import os

n_workers = int(os.environ.get("SLURM_CPUS_PER_TASK", "1"))

with ProcessPoolExecutor(max_workers=n_workers) as pool:
    results = list(pool.map(run_one_task, tasks))
```

Avoid nested parallelism: Slurm array × Python multiprocessing × BLAS threads can explode CPU usage.

## Resumable numbered tasks

Write scripts so each Slurm array task can be restarted safely. If the output for task 17 exists, task 17 exits without doing work.

```python
import argparse
from pathlib import Path

import polars as pl

parser = argparse.ArgumentParser()
parser.add_argument("--task-id", type=int, required=True)
args = parser.parse_args()

output = Path(f"/gpfs/project/myproject/output/task_{args.task_id:04d}.parquet")
if output.exists():
    print(f"task {args.task_id} already done: {output}")
    raise SystemExit(0)

# Replace this with real task-specific work.
result = pl.DataFrame({"task_id": [args.task_id], "value": [args.task_id ** 2]})

tmp = output.with_suffix(".parquet.tmp")
result.write_parquet(tmp)
tmp.rename(output)
```

This pattern lets you rerun the same array and only compute missing tasks.

## GPU check

Only use this in a GPU allocation:

```python
import torch

assert torch.cuda.is_available(), "No CUDA device visible"
print(torch.cuda.get_device_name(0))
```

## Checklist

- [ ] Project uses `uv` and committed lockfile.
- [ ] Slurm script sets thread variables.
- [ ] Python reads `SLURM_CPUS_PER_TASK` with default `"1"`.
- [ ] Multiprocessing workers do not exceed allocated CPUs.
- [ ] Outputs are skip-if-exists and atomically written.
- [ ] Logs include job ID and key progress messages.

## Further reading

- [uv documentation](https://docs.astral.sh/uv/) — projects, environments, lockfiles, `uv run`.
- [Slurm sbatch reference](https://slurm.schedmd.com/sbatch.html) — directives and `SLURM_*` env vars Python reads.
- [`logging` module](https://docs.python.org/3/library/logging.html) — handlers, formatters, levels.
- [`argparse` module](https://docs.python.org/3/library/argparse.html) — CLI args for resumable task scripts.
