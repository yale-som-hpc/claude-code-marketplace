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
updated: 2026-04-29
---
# Running Python

Rule: use a project environment, control threads, log clearly, and make outputs resumable.

## Tooling defaults

These are slightly opinionated picks. Each has a one-line "why on the cluster."

- **`uv`** for dependencies, lockfiles, and Python interpreter management. Faster than conda on GPFS, and the lockfile reproduces on a compute node.
- **`ruff`** for lint + format. Catches mistakes locally before burning a Slurm allocation.
- **`pyrefly`** for type checking. Same reason as ruff.
- **`pytest`** for tests. Smoke tests on small inputs save many cluster reruns.
- **`argparse`** for batch scripts (one-file entry points like `run_task.py --task-id`). `click` only when you grow into a reusable project CLI; the extra dependency is not worth it for a single sbatch script.
- **`pathlib`** over `os.path`. Joining paths and checking parents is what you do most on the cluster.
- **`logging`** as the baseline (configured below). `loguru` is fine when its structured output materially helps incident debugging.
- **`pyproject.toml` + `uv.lock` committed; `.venv/` gitignored.** The lockfile is what makes runs reproducible across login and compute nodes.

How to install the tools themselves: see [installing software](../installing-software/SKILL.md) for `uv` (one curl command into `~/.local/bin`). Once `uv` is on your PATH, `ruff`, `pyrefly`, and `pytest` go in your project's dev dependencies via `uv add --dev ruff pyrefly pytest`, so they reproduce from `uv.lock` like everything else.

## Project setup with uv

```bash
cd /gpfs/project/myproject/code
uv init --app
uv add polars pyarrow duckdb
uv sync --frozen
```

This is a setup-time operation, run once on a login node. Do not run `uv sync` inside Slurm jobs or job arrays — environment mutation in flight is a waste pattern (and `--frozen` makes it explicit that the lockfile is the source of truth).

Commit:

```text
pyproject.toml
uv.lock
```

Do not commit `.venv/`; put it in `.gitignore`.

Avoid `pip install --user` and `pip install` inside jobs. They are not reproducible and they leak state between projects. If you need a one-off package, `uv add` and `uv sync --frozen` is the path. (Pip itself is fine; the bad defaults are `--user` and per-job installs.) See [installing software](../installing-software/SKILL.md) for the broader picture.

## Data work, default picks

For most cluster work, these are the right defaults:

- **Tabular reads/writes** → [Polars](https://pola.rs/) for new code (uses your CPU allocation through threading and lazy `scan_*`); pandas at API boundaries (sklearn, statsmodels, plotting). Convert with `.to_pandas()` only at the boundary; round-tripping doubles memory.
- **File format** → Parquet with `compression="zstd"`. One reused Parquet beats 10k CSVs both for speed and for GPFS metadata health.
- **Lazy reads** → `pl.scan_csv` / `pl.scan_parquet` push filters and column projection before materialization, keeping memory under your `--mem` limit.
- **Append-heavy / streaming output** → JSONL with gzip is the simplest correct option for record-by-record writes (one append-only file, atomic at line granularity). For columnar appends, write **one Parquet per task or per chunk** (`out/task_0001.parquet`, `out/task_0002.parquet`, …) and read them back with `pl.scan_parquet("out/*.parquet")`. Do not mutate one big Parquet in place. See [using the filesystem](../using-the-filesystem/SKILL.md) for the full append patterns.
- **SQL over local files** → DuckDB. Joins CSV/Parquet/JSON without staging.
- **Local cache / lookup tables** → SQLite with `PRAGMA journal_mode=WAL`; one connection per process; keep writes single-writer.
- **Multi-user database** → `psycopg` with `psycopg_pool`; create one pool per process if you fork.
- **Unknown encodings** → `charset-normalizer` to detect, then pass `encoding=` explicitly.

Worked examples for query patterns and ingestion live in [working with large data](../working-with-large-data/SKILL.md) and [acquiring data](../acquiring-data/SKILL.md).

## Safe Python Slurm template

Use this shape as the default. The launch line is `srun .venv/bin/python ...`, not `uv run python ...`, so SIGUSR1 reaches Python on long jobs (see [parallel Python](../parallel-python/SKILL.md) for why):

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
export PYTHONUNBUFFERED=1                              # Slurm logs update during the job, not just at exit

cd /gpfs/project/myproject/code
# Environment was created during setup with: uv sync --frozen
srun .venv/bin/python src/main.py
```

For short, exploratory one-off jobs where signal-based shutdown does not matter, `uv run python src/main.py` (without `srun`) is acceptable. For anything long-running or resumable, use the `srun .venv/bin/python` form.

## Read Slurm settings safely

`SLURM_*` env vars are only set inside Slurm jobs. The patterns below let the same script run on your laptop (no Slurm) and on the cluster (Slurm fills in real values):

```python
import os

# Slurm allocation when running under sbatch/srun, fall back to the
# laptop's CPU count for local testing.
n_cpus = int(os.environ.get("SLURM_CPUS_PER_TASK", "0")) or os.cpu_count() or 1

# "local" is a useful sentinel for log file names off-cluster.
job_id = os.environ.get("SLURM_JOB_ID", "local")
```

The first line reads as "use Slurm's CPU count if it's set, else `os.cpu_count()`, else 1." On the cluster you get the allocation; on a laptop you get the local CPU count; in a constrained container you still get a sensible non-zero number.

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

Set `PYTHONUNBUFFERED=1` in the Slurm script (above) so log lines reach `logs/*.out` while the job is running, not all at once at the end. Use logs to know what happened without opening notebooks.

## Multiprocessing

For nontrivial multiprocessing, producer-consumer queues, process pools, or job-stealing-style work, use [parallel Python](../parallel-python/SKILL.md).

Minimal rule: match worker count to allocated CPUs.

```python
from concurrent.futures import ProcessPoolExecutor
import os

n_workers = int(os.environ.get("SLURM_CPUS_PER_TASK", "0")) or os.cpu_count() or 1

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
    print(f"task {args.task_id} already done: {output}", flush=True)
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

- [ ] Project uses `uv` and committed `uv.lock`.
- [ ] `uv sync --frozen` runs at setup, never inside arrays or jobs.
- [ ] Slurm script sets `OMP/MKL/OPENBLAS/NUMEXPR_NUM_THREADS` and `PYTHONUNBUFFERED=1`.
- [ ] Long or resumable jobs launch with `srun .venv/bin/python ...`, not `uv run python ...`, so SIGUSR1 reaches Python.
- [ ] Python reads `SLURM_CPUS_PER_TASK` with a laptop-friendly fallback (`os.cpu_count()`).
- [ ] Multiprocessing workers do not exceed allocated CPUs.
- [ ] Outputs are skip-if-exists and atomically written.
- [ ] Logs include job ID and key progress messages.

## Further reading

- [uv documentation](https://docs.astral.sh/uv/) — projects, environments, lockfiles, `uv run`.
- [ruff](https://docs.astral.sh/ruff/) and [pyrefly](https://github.com/facebook/pyrefly) — lint/format and type checking.
- [Polars user guide](https://docs.pola.rs/) — eager and lazy DataFrames, `scan_*`, expressions.
- [DuckDB Python API](https://duckdb.org/docs/api/python/overview) — SQL over local files.
- [psycopg 3](https://www.psycopg.org/psycopg3/docs/) and [psycopg_pool](https://www.psycopg.org/psycopg3/docs/advanced/pool.html) — PostgreSQL with pooled connections.
- [Slurm sbatch reference](https://slurm.schedmd.com/sbatch.html) — directives and `SLURM_*` env vars Python reads.
- [`logging` module](https://docs.python.org/3/library/logging.html) — handlers, formatters, levels.
- [`argparse` module](https://docs.python.org/3/library/argparse.html) — CLI args for resumable task scripts.
