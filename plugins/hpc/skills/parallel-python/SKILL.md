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
updated: 2026-04-29
---
# Parallel Python

Rule: pick one main layer of parallelism, match it to the bottleneck, and size workers from the Slurm allocation. If one process is too slow, first check [accelerating Python](../accelerating-python/SKILL.md).

## Choose the right tool

| Workload | Use | Avoid |
|---|---|---|
| Independent tasks across files/years/specs | Slurm array or GNU parallel | One giant Python process |
| CPU-bound pure Python | `ProcessPoolExecutor` or `multiprocessing` | Threads, because of the GIL |
| I/O-bound HTTP/API/database work | threads or async | Too many processes |
| Variable-duration tasks | `imap_unordered` or `as_completed` | Static equal chunks |
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

## Three rungs of parallelism

Climb only as high as you need. Most cluster jobs live on rung 1.

### Rung 1: `ProcessPoolExecutor` for CPU-bound work

Submit independent tasks; log from the parent as each one finishes. The parent is the only writer, so logs stay readable and shutdown is straightforward.

```python
from concurrent.futures import ProcessPoolExecutor, as_completed
import logging
import os

logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(message)s")
log = logging.getLogger("driver")

n_workers = int(os.environ.get("SLURM_CPUS_PER_TASK", "1"))

def work(task_id: int) -> tuple[int, int]:
    return task_id, task_id ** 2

with ProcessPoolExecutor(max_workers=n_workers) as pool:
    futures = {pool.submit(work, task_id): task_id for task_id in range(1000)}
    for future in as_completed(futures):
        task_id, value = future.result()
        log.info("done task=%d value=%d", task_id, value)
```

`as_completed` keeps fast workers taking new work while slow ones finish.

#### Expensive worker setup

If each task needs a multi-GB model or large lookup table, do not load it inside `work()` — that loads it once per task. And do not load it in the parent before fanning out — that copies it (with `fork`) or reloads it 16 times into RAM and risks an OOM kill.

Use a per-process initializer so each worker loads it once:

```python
_model = None

def init_worker():
    global _model
    _model = load_model_once()

def classify(batch):
    return _model.predict(batch)

with ProcessPoolExecutor(max_workers=n_workers, initializer=init_worker) as pool:
    for result in pool.map(classify, batches, chunksize=16):
        write_result(result)
```

### Rung 2: `imap_unordered` for uneven task durations

When tasks vary by 10× or more in runtime, `imap_unordered(..., chunksize=1)` is a simple work-stealing pattern:

```python
from multiprocessing import Pool
import os

n_workers = int(os.environ.get("SLURM_CPUS_PER_TASK", "1"))

with Pool(processes=n_workers) as pool:
    for done_path in pool.imap_unordered(process_file, input_paths, chunksize=1):
        print(f"done {done_path}", flush=True)
```

For uniform work, start with `chunksize=len(items) // n_workers`. For uneven work, use `chunksize=1`. Larger `chunksize` only helps when each task is tiny and dispatch overhead dominates.

### Rung 3: Bounded queue with a single writer

Use this only when you genuinely need a staged pipeline: one stage reads, workers transform, one writer serializes. Bounded queues bound memory; a single writer avoids interleaved output and the metadata cost of every worker opening its own files.

This is a recipe — copy the whole shape. The `task_done()` calls, the one-sentinel-per-worker count, the bounded `output_queue`, and starting the writer *before* the workers are all load-bearing. Missing any of them is a common cause of jobs hanging until `SIGKILL`.

```python
from multiprocessing import JoinableQueue, Process, Queue
import os

SENTINEL = None

def worker(input_queue: JoinableQueue, output_queue: Queue):
    while True:
        item = input_queue.get()
        try:
            if item is SENTINEL:
                return
            output_queue.put(transform(item))
        finally:
            input_queue.task_done()  # required even for the sentinel

def writer(output_queue: Queue):
    while True:
        item = output_queue.get()
        if item is SENTINEL:
            return
        write_results(item)  # one process owns all output

n_workers = int(os.environ.get("SLURM_CPUS_PER_TASK", "1"))
input_queue = JoinableQueue(maxsize=n_workers * 3)   # bounded → memory-bounded
output_queue = Queue(maxsize=n_workers * 3)          # bounded → workers cannot outrun the writer

writer_proc = Process(target=writer, args=(output_queue,))
writer_proc.start()                                   # writer must be running before workers produce

workers = [
    Process(target=worker, args=(input_queue, output_queue))
    for _ in range(n_workers)
]
for p in workers:
    p.start()

for item in tasks:
    input_queue.put(item)
for _ in workers:                                     # one sentinel per worker
    input_queue.put(SENTINEL)

input_queue.join()                                    # wait for all task_done() calls

for p in workers:
    p.join(timeout=30)
    if p.is_alive():
        p.terminate()

output_queue.put(SENTINEL)                            # only after workers have stopped producing
writer_proc.join(timeout=30)
if writer_proc.is_alive():
    writer_proc.terminate()
```

Use threads (`queue.Queue`, `threading.Thread`) instead of processes when the workload is I/O-bound; the shape of the recipe is the same.

## Logging from worker processes

Log from the parent first. With `as_completed` (rung 1) or `imap_unordered` (rung 2), the parent sees every task's return value, so you can log without ever logging from a worker process.

When workers must log directly:

- **One log file per worker** is the simplest correct option. Open the file inside `init_worker()` keyed by worker rank or PID.
- **`QueueHandler` + `QueueListener`** when workers must share one log stream. Records funnel through the listener so they don't interleave:

```python
import logging
import multiprocessing as mp
from logging.handlers import QueueHandler, QueueListener

log_queue = mp.Queue()
listener = QueueListener(log_queue, logging.StreamHandler())
listener.start()

# In each worker (set up inside init_worker for spawned workers):
logger = logging.getLogger("worker")
logger.addHandler(QueueHandler(log_queue))
logger.setLevel(logging.INFO)
```

Multiple processes writing to the same stream without funneling produce garbled lines that defeat `seff` and post-mortem debugging.

## Debug stuck jobs

Long Slurm jobs that hang are expensive — they burn allocation until the time limit. Make stuck jobs diagnosable up front:

```python
import faulthandler
import signal
import sys

faulthandler.enable()                                       # SIGSEGV/SIGABRT/SIGFPE dump tracebacks
faulthandler.register(signal.SIGUSR2, file=sys.stderr, all_threads=True)
```

Then from another shell on the same compute node:

```bash
kill -USR2 <pid>          # dump every thread's stack to stderr
```

`SIGUSR1` is reserved for time-limit shutdown (below); `SIGUSR2` carries the live stack dump. `py-spy dump --pid <pid>` is even better when available; it works without changing your code. Log queue depth, worker start/finish, and output counts at INFO — a silent pipeline is painful to debug.

## When to use Slurm arrays instead

If tasks are independent and each task takes minutes or hours, use a Slurm array, not Python multiprocessing:

```bash
#SBATCH --array=1-1000%50
srun .venv/bin/python src/run_task.py --task-id "${SLURM_ARRAY_TASK_ID}"
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

Pair with `#SBATCH --signal=USR1@300` for a 5-minute warning before the time limit, and with skip-if-exists outputs so reruns pick up where the last job left off.

The launch line must let SIGUSR1 reach the Python process. Tested on this cluster:

```bash
#SBATCH --signal=USR1@300
# Environment was built during setup with: uv sync --frozen
srun .venv/bin/python src/main.py
```

These do **not** reliably deliver SIGUSR1 to Python and will SIGKILL your job at the time limit instead:

- `#SBATCH --signal=B:USR1@300` — sends to the batch shell only; bash dies, Python never sees it.
- `python src/main.py` directly under bash (no `srun`) — the signal does not propagate from the batch shell to the Python child.
- `srun uv run python src/main.py` — `uv run` is the foreground process; it has no SIGUSR1 handler and dies on the default action before Python can react.

Run Python as the `srun` step directly (`srun .venv/bin/python ...`) so the signal lands on Python.

## Checklist

- [ ] Bottleneck is known: CPU, I/O, database, network, GPU, or filesystem.
- [ ] Worker count comes from `SLURM_CPUS_PER_TASK`.
- [ ] Only one main layer of parallelism; nested layers are pinned to 1.
- [ ] `spawn` is used when CUDA, threads, or open handles are involved.
- [ ] Expensive per-worker state is loaded in an `initializer`, not on every task.
- [ ] Queue pipelines use bounded queues, one sentinel per worker, `task_done()` in `finally`, and one writer started before workers.
- [ ] Logging happens in the parent unless workers genuinely need their own stream.
- [ ] `faulthandler.enable()` is called at startup; `SIGUSR2` is registered for live stack dumps on long jobs.
- [ ] Independent multi-minute tasks use Slurm arrays, not multiprocessing.
- [ ] Long or resumable jobs use `#SBATCH --signal=USR1@300` and launch with `srun .venv/bin/python ...` so SIGUSR1 reaches Python.
- [ ] Outputs are written incrementally and atomically so time-limit kills are recoverable.

## Further reading

- [`multiprocessing`](https://docs.python.org/3/library/multiprocessing.html) — `Pool`, `Process`, queues, locks.
- [Start methods](https://docs.python.org/3/library/multiprocessing.html#contexts-and-start-methods) — `spawn`, `fork`, `forkserver` semantics.
- [`concurrent.futures`](https://docs.python.org/3/library/concurrent.futures.html) — `ProcessPoolExecutor`, `as_completed`, `map`.
- [`logging.handlers.QueueListener`](https://docs.python.org/3/library/logging.handlers.html#queuelistener) — process-safe logging.
- [`faulthandler`](https://docs.python.org/3/library/faulthandler.html) — stack dumps on crash or signal.
- [py-spy](https://github.com/benfred/py-spy) — sampling profiler and live `dump` for stuck processes.
- [joblib](https://joblib.readthedocs.io/en/stable/) — `Parallel`/`delayed`, memoization.
- [Dask](https://docs.dask.org/en/stable/) and [Ray](https://docs.ray.io/en/latest/) — when one node isn't enough.
- [Slurm job arrays](https://slurm.schedmd.com/job_array.html) — for embarrassingly-parallel tasks at the scheduler layer.
- [Slurm `--signal`](https://slurm.schedmd.com/sbatch.html#OPT_signal) — signal delivery semantics; the `B:` prefix routes to the batch shell only.
