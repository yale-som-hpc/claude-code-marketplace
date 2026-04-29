---
name: parallel-python
description: Choose threads, processes, queues, and Slurm arrays for Python on the Yale SOM HPC cluster. TRIGGER when running multiprocessing/concurrent.futures/joblib/Dask/Ray Python work on the Yale SOM HPC cluster, writing Slurm array jobs there, or right-sizing workers to SLURM_CPUS_PER_TASK on the cluster.
related:
  - running-python
  - accelerating-python
  - managing-jobs
  - working-with-large-data
  - acquiring-data
  - self-diagnosing-resource-use
updated: 2026-04-28
---
# Parallel Python

Rule: pick one main layer of parallelism, match it to the bottleneck, and size workers from the Slurm allocation. If one process is too slow, first check [accelerating Python](../accelerating-python/SKILL.md).

## Choose the right tool

| Workload | Use | Avoid |
|---|---|---|
| Independent tasks across files/years/specs | Slurm array or GNU parallel | One giant Python process |
| CPU-bound pure Python | `ProcessPoolExecutor` or `multiprocessing` | Threads, because of the GIL |
| I/O-bound HTTP/API/database work | threads or async | Too many processes |
| Variable-duration tasks | `imap_unordered` or queue workers | Static equal chunks |
| Large numeric kernels | BLAS/NumPy/Polars/DuckDB threads | Extra Python processes unless needed |
| GPU training/inference | one process per GPU pattern | CPU preprocessing inside GPU allocation |

## Respect the Slurm allocation

```python
import os

n_workers = int(os.environ.get("SLURM_CPUS_PER_TASK", "1"))
```

If Slurm gives you 4 CPUs, do not start 32 workers. If each worker calls BLAS/NumPy, also set thread environment variables in the Slurm script.

## Avoid nested parallelism

This is the single biggest waste pattern on the cluster.

Bad:

```text
500 Slurm array tasks × 16 Python processes × 8 BLAS threads = 64,000 runnable threads
```

Good:

```text
50 concurrent Slurm tasks × 1 Python process × 1 BLAS thread
```

or:

```text
1 Slurm job × 16 Python processes × 1 BLAS thread
```

Pick one main layer of parallelism. If you genuinely need two layers, document why and pin the others to 1.

## Start methods (CUDA, threads, open handles)

Choose multiprocessing start methods explicitly with `get_context()`:

```python
import multiprocessing as mp

ctx = mp.get_context("spawn")
```

Use:

- `spawn` for CUDA/PyTorch or when the parent has active threads/open handles.
- `forkserver` as a safer general-purpose alternative to `fork`.
- `fork` only when you understand the inherited state and do not use CUDA.

`fork` + CUDA is a common cause of mysteriously hung GPU jobs.

## Dynamic load balancing for uneven work

`imap_unordered(..., chunksize=1)` is a simple work-stealing pattern when task durations vary:

```python
from multiprocessing import Pool
import os

n_workers = int(os.environ.get("SLURM_CPUS_PER_TASK", "1"))

with Pool(processes=n_workers) as pool:
    for done_path in pool.imap_unordered(process_file, input_paths, chunksize=1):
        print(f"done {done_path}", flush=True)
```

For uniform work, start with `len(items) // n_workers`; for uneven work, use smaller chunks.

## When to use Slurm arrays instead

If tasks are independent and each task takes minutes or hours, use a Slurm array, not Python multiprocessing:

```bash
#SBATCH --array=1-1000%50
uv run python src/run_task.py --task-id "${SLURM_ARRAY_TASK_ID}"
```

Slurm gets visibility into the work, failed tasks are easy to rerun, and one bad task does not take down the whole job.

## Handle time-limit interruption

Slurm's `KillWait=30` means ~30 seconds between final `SIGTERM` and `SIGKILL`. Design for frequent atomic output and fast shutdown.

```python
import signal
import threading

stop_requested = threading.Event()

def request_stop(signum, frame):
    stop_requested.set()

signal.signal(signal.SIGTERM, request_stop)
signal.signal(signal.SIGUSR1, request_stop)

for task in tasks:
    if stop_requested.is_set():
        break
    run_task(task)
    write_atomic_result(task)
```

Pair with `#SBATCH --signal=B:USR1@300` for a 5-minute warning before the time limit, and with skip-if-exists outputs so reruns pick up where the last job left off.

## Checklist

- [ ] Worker count comes from `SLURM_CPUS_PER_TASK`.
- [ ] Only one main layer of parallelism; nested layers are pinned to 1.
- [ ] `spawn` is used when CUDA, threads, or open handles are involved.
- [ ] Independent multi-minute tasks use Slurm arrays, not multiprocessing.
- [ ] Outputs are written incrementally and atomically so time-limit kills are recoverable.

## Further reading

- [`multiprocessing`](https://docs.python.org/3/library/multiprocessing.html) — `Pool`, `Process`, queues, locks.
- [Start methods](https://docs.python.org/3/library/multiprocessing.html#contexts-and-start-methods) — `spawn`, `fork`, `forkserver` semantics.
- [`concurrent.futures`](https://docs.python.org/3/library/concurrent.futures.html) — `ProcessPoolExecutor`, `as_completed`, `map`.
- [joblib](https://joblib.readthedocs.io/en/stable/) — `Parallel`/`delayed`, memoization.
- [Dask](https://docs.dask.org/en/stable/) and [Ray](https://docs.ray.io/en/latest/) — when one node isn't enough.
- [Slurm job arrays](https://slurm.schedmd.com/job_array.html) — for embarrassingly-parallel tasks at the scheduler layer.
