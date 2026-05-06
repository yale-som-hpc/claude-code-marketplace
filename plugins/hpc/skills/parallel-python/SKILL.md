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

`SLURM_CPUS_PER_TASK` is set inside Slurm jobs and unset on your laptop. Use a small helper so the same script works in both places:

```python
import os

def available_workers() -> int:
    """Match the Slurm allocation when running under sbatch/srun, fall back
    to the local machine's CPU count for laptop / login-node testing."""
    slurm = os.environ.get("SLURM_CPUS_PER_TASK")
    if slurm:
        return int(slurm)
    return os.cpu_count() or 1
```

If Slurm gives you 4 CPUs, do not start 32 workers. If each worker calls BLAS/NumPy, also set thread environment variables in the Slurm script.

## Avoid nested parallelism

This is the single biggest waste pattern on the cluster.

Bad: 500 Slurm array tasks × 16 Python processes × 8 BLAS threads = 64,000 runnable threads.

Good: 50 concurrent Slurm tasks × 1 Python process × 1 BLAS thread; or: 1 Slurm job × 16 Python processes × 1 BLAS thread

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

n_workers = available_workers()  # see "Respect the Slurm allocation" above

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

n_workers = available_workers()  # see "Respect the Slurm allocation" above

with Pool(processes=n_workers) as pool:
    for done_path in pool.imap_unordered(process_file, input_paths, chunksize=1):
        print(f"done {done_path}", flush=True)
```

For uniform work, start with `chunksize=len(items) // n_workers`. For uneven work, use `chunksize=1`. Larger `chunksize` only helps when each task is tiny and dispatch overhead dominates.

### Rung 3: Bounded queue with a single writer

Use this only when you genuinely need a staged pipeline: one stage reads, workers transform, one writer serializes. Bounded queues bound memory; a single writer avoids interleaved output and the metadata cost of every worker opening its own files.

This is a recipe — copy the whole shape. The `task_done()` calls, the one-quit-signal-per-worker count, the bounded `output_queue`, and starting the writer *before* the workers are all load-bearing. Missing any of them is a common cause of jobs hanging until `SIGKILL`.

```python
from multiprocessing import JoinableQueue, Process, Queue
import os

QUIT_SIGNAL = None

def worker(input_queue: JoinableQueue, output_queue: Queue):
    while True:
        item = input_queue.get()
        try:
            if item is QUIT_SIGNAL:
                return
            output_queue.put(transform(item))
        finally:
            input_queue.task_done()  # required even for the quit signal

def writer(output_queue: Queue):
    while True:
        item = output_queue.get()
        if item is QUIT_SIGNAL:
            return
        write_results(item)  # one process owns all output

n_workers = available_workers()  # see "Respect the Slurm allocation" above
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
for _ in workers:                                     # one quit signal per worker
    input_queue.put(QUIT_SIGNAL)

input_queue.join()                                    # wait for all task_done() calls

for p in workers:
    p.join(timeout=30)
    if p.is_alive():
        p.terminate()

output_queue.put(QUIT_SIGNAL)                            # only after workers have stopped producing
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

A Python job that hangs on the cluster is genuinely expensive: Slurm keeps the allocation reserved until the time limit, blocking other people's jobs. Worse, when it eventually times out, you may have no idea where it was stuck. Two small additions to your script make stuck jobs diagnosable instead of silent.

`faulthandler` is in the Python standard library — no extra install. It does two useful things on the cluster:

1. **`faulthandler.enable()`** — when Python crashes hard (segfault from a C extension, division-by-zero in NumPy with certain flags, etc.), Python normally just dies with no traceback. With `faulthandler.enable()`, it prints the Python traceback to stderr first, so you actually see where the crash happened.

2. **`faulthandler.register(signal.SIGUSR2, ...)`** — turns a signal into a "dump every thread's stack to stderr right now" command. Useful when your job is *running* but stuck (deadlock, infinite loop, waiting on a network call), and you want to see what every worker thread is doing without restarting the program.

Add this at the top of any long-running script:

```python
import faulthandler
import signal
import sys

faulthandler.enable()                                                  # crash-time tracebacks
faulthandler.register(signal.SIGUSR2, file=sys.stderr, all_threads=True)  # on-demand stack dump
```

To trigger the on-demand dump from another shell on the same compute node:

```bash
# From another login shell, ssh to the compute node first:
ssh c019                  # or whichever node Slurm placed your job on
kill -USR2 <python-pid>   # find the PID with: ps -fu $USER | grep python
```

Why `SIGUSR2` and not `SIGUSR1`? `SIGUSR1` is the signal we use for time-limit shutdown (see "Handle time-limit interruption" below). Splitting them means you can request a stack dump *and* later request a clean shutdown without the two getting confused.

Aside from `faulthandler`, the cheapest debug tool is logging. Log queue depth, worker start/finish, and output counts at INFO. A silent pipeline is painful to debug.

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

### Why `srun .venv/bin/python` and not `uv run python`

The launch line has to let SIGUSR1 reach the Python process. Tested on `default_queue` with a `uv sync --frozen` venv:

| Launch line | What happens to SIGUSR1 | Result |
|---|---|---|
| `srun .venv/bin/python script.py` | Slurm delivers it to the srun step → Python's handler fires | **Clean shutdown** |
| `srun uv run python script.py` | `uv run` is the foreground process and installs no SIGUSR1 handler — dies on the default action before exec'ing through to Python | Job FAILED |
| `python script.py` (no `srun`) | Slurm signals the step but bash + child Python doesn't propagate to the child | TIMEOUT |
| `#SBATCH --signal=B:USR1@N` | The `B:` prefix routes only to the batch shell; bash dies on the default action and kills Python | Job FAILED |

So:

```bash
#SBATCH --signal=USR1@300                                           # not B:USR1@300
# Environment was built during setup with: uv sync --frozen
srun .venv/bin/python src/main.py                                   # not uv run python
```

For short exploratory runs that won't hit the time limit, `uv run python` is fine — see [running Python](../running-python/SKILL.md#safe-python-slurm-template).

## Checklist

- [ ] Bottleneck is known: CPU, I/O, database, network, GPU, or filesystem.
- [ ] Worker count comes from `SLURM_CPUS_PER_TASK` with a laptop-friendly fallback (`os.cpu_count()`).
- [ ] Only one main layer of parallelism; nested layers are pinned to 1.
- [ ] `spawn` is used when CUDA, threads, or open handles are involved.
- [ ] Expensive per-worker state is loaded in an `initializer`, not on every task.
- [ ] Queue pipelines use bounded queues, one quit signal per worker, `task_done()` in `finally`, and one writer started before workers.
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
- [joblib](https://joblib.readthedocs.io/en/stable/) — `Parallel`/`delayed`, memoization.
- [Dask](https://docs.dask.org/en/stable/) and [Ray](https://docs.ray.io/en/latest/) — when one node isn't enough.
- [Slurm job arrays](https://slurm.schedmd.com/job_array.html) — for embarrassingly-parallel tasks at the scheduler layer.
- [Slurm `--signal`](https://slurm.schedmd.com/sbatch.html#OPT_signal) — signal delivery semantics; the `B:` prefix routes to the batch shell only.
