---
title: Parallel Python
slug: parallel-python
description: Choose threads, processes, queues, job arrays, and shutdown patterns for Python work on the cluster. TRIGGER when using multiprocessing, concurrent.futures, joblib, Dask/Ray, Slurm arrays, or any Python parallel workload.
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

Rule: match the parallelism model to the bottleneck. Do not add processes, threads, and Slurm arrays on top of each other without a reason. If one process is simply too slow, first check [accelerating Python](./accelerating-python.md).

## Choose the right tool

| Workload | Use | Avoid |
|---|---|---|
| Independent tasks across files/years/specs | Slurm array or GNU parallel | One giant Python process |
| CPU-bound pure Python | `ProcessPoolExecutor` or `multiprocessing` | Threads, because of the GIL |
| I/O-bound HTTP/API/database work | threads or async | Too many processes |
| Variable-duration tasks | queue workers or `imap_unordered` | Static equal chunks |
| Large numeric kernels | BLAS/NumPy/Polars/DuckDB threads | Extra Python processes unless needed |
| GPU training/inference | one process per GPU pattern | CPU preprocessing inside GPU allocation |

Start with the simplest model that keeps resources busy.

## Respect the Slurm allocation

```python
import os

n_workers = int(os.environ.get("SLURM_CPUS_PER_TASK", "1"))
```

If Slurm gives you 4 CPUs, do not start 32 workers. If each worker calls BLAS/NumPy, also set thread environment variables in the Slurm script.

## Start methods

Choose multiprocessing start methods explicitly with `get_context()`.

```python
import multiprocessing as mp

ctx = mp.get_context("spawn")
queue = ctx.Queue()
process = ctx.Process(target=worker, args=(queue,))
```

Use:

- `spawn` for CUDA/PyTorch or when parent process has active threads/open handles.
- `forkserver` as a safer general-purpose alternative to `fork`.
- `fork` only when you understand the inherited state and do not use CUDA or active threads.

Prefer `get_context()` over global `set_start_method()` so libraries and tests stay composable.

## Safe process pool

Use processes for CPU-bound Python functions:

```python
from concurrent.futures import ProcessPoolExecutor, as_completed
import os

n_workers = int(os.environ.get("SLURM_CPUS_PER_TASK", "1"))

def work(task_id: int) -> tuple[int, int]:
    return task_id, task_id ** 2

with ProcessPoolExecutor(max_workers=n_workers) as pool:
    futures = [pool.submit(work, task_id) for task_id in range(1000)]
    for future in as_completed(futures):
        task_id, value = future.result()
        print(task_id, value)
```

Use `as_completed` when tasks have uneven runtimes so fast workers keep taking new work.

## Dynamic load balancing

For many uneven tasks, `imap_unordered(..., chunksize=1)` is a simple work-stealing-like pattern:

```python
from multiprocessing import Pool
import os

n_workers = int(os.environ.get("SLURM_CPUS_PER_TASK", "1"))

def process_file(path: str) -> str:
    # Parse, transform, write output, return status.
    return path

with Pool(processes=n_workers) as pool:
    for done_path in pool.imap_unordered(process_file, input_paths, chunksize=1):
        print(f"done {done_path}", flush=True)
```

Use larger `chunksize` only when each task is tiny and overhead dominates. For uniform work, start with `len(items) // n_workers`; for uneven work, use smaller chunks.

If worker setup is expensive, use a per-process initializer:

```python
_model = None

def init_worker():
    global _model
    _model = load_model_once()

with ProcessPoolExecutor(max_workers=n_workers, initializer=init_worker) as pool:
    for result in pool.map(classify_with_global_model, batches, chunksize=16):
        write_result(result)
```

## Producer-consumer pipeline

Use this when one stage reads, workers transform, and one writer serializes results. This pattern avoids concurrent writes and bounds memory.

```python
from queue import Queue
from threading import Thread

SENTINEL = object()

def producer(input_queue: Queue):
    for batch in read_batches(batch_size=100):
        input_queue.put(batch)
    input_queue.put(SENTINEL)

def worker(worker_id: int, input_queue: Queue, output_queue: Queue):
    while True:
        batch = input_queue.get()
        if batch is SENTINEL:
            input_queue.put(SENTINEL)  # pass shutdown to next worker
            output_queue.put(SENTINEL)
            return
        output_queue.put(transform(batch))

def writer(output_queue: Queue, n_workers: int):
    finished = 0
    while finished < n_workers:
        item = output_queue.get()
        if item is SENTINEL:
            finished += 1
            continue
        write_results(item)

n_workers = 4
input_queue = Queue(maxsize=n_workers * 3)
output_queue = Queue()

threads = [Thread(target=worker, args=(i, input_queue, output_queue)) for i in range(n_workers)]
for t in threads:
    t.start()

Thread(target=producer, args=(input_queue,)).start()
writer(output_queue, n_workers)
for t in threads:
    t.join()
```

Use threads for I/O-bound calls. For CPU-bound transformations, use `multiprocessing.Queue` and `multiprocessing.Process`, but keep the same bounded-queue and single-writer shape.

## Multiprocessing queue shutdown

For process workers, send one poison pill per worker and pair `get()` with `task_done()` when using `JoinableQueue`.

```python
from multiprocessing import JoinableQueue, Process

SENTINEL = None

def worker(input_queue: JoinableQueue):
    while True:
        item = input_queue.get()
        try:
            if item is SENTINEL:
                return
            process(item)
        finally:
            input_queue.task_done()

workers = [Process(target=worker, args=(input_queue,)) for _ in range(n_workers)]
for p in workers:
    p.start()

for item in tasks:
    input_queue.put(item)
for _ in workers:
    input_queue.put(SENTINEL)

input_queue.join()
for p in workers:
    p.join(timeout=30)
    if p.is_alive():
        p.terminate()
```

Missing `task_done()` is a common cause of jobs hanging forever in `join()`.

## Process-safe logging

Python logging is thread-safe, but multiple processes writing to the same stream or file can garble logs. Funnel records through one listener.

```python
import logging
import multiprocessing as mp
from logging.handlers import QueueHandler, QueueListener

log_queue = mp.Queue()
listener = QueueListener(log_queue, logging.StreamHandler())
listener.start()

logger = logging.getLogger("worker")
logger.addHandler(QueueHandler(log_queue))
logger.setLevel(logging.INFO)
```

Set up the `QueueHandler` inside each worker if workers are spawned.

## Debug stuck jobs

Enable fault dumps early:

```python
import faulthandler

faulthandler.enable()
```

If allowed, attach to a running Python process:

```bash
py-spy dump --pid PID
```

Use logs for queue depth, worker start/finish, and output counts. A silent queue pipeline is painful to debug.

## Handle interruption

Slurm may kill jobs at the time limit. On this cluster, the final grace period is short, so design for frequent output and fast shutdown.

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

For long jobs, pair this with `#SBATCH --signal=B:USR1@300` and resumable outputs.

## When to use Slurm arrays instead

If tasks are independent and each task takes minutes or hours, use a Slurm array instead of Python multiprocessing:

```bash
#SBATCH --array=1-1000%50
uv run python src/run_task.py --task-id "${SLURM_ARRAY_TASK_ID}"
```

This gives Slurm visibility into the work, makes failed tasks easy to rerun, and avoids one giant fragile process.

## Avoid nested parallelism

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

Choose one main layer of parallelism.

## Checklist

- [ ] Bottleneck is known: CPU, I/O, database, network, GPU, or filesystem.
- [ ] Worker count comes from `SLURM_CPUS_PER_TASK`.
- [ ] Queues are bounded when producers can outrun consumers.
- [ ] One writer owns output serialization.
- [ ] Shutdown uses sentinels or signal flags, not abandoned workers.
- [ ] Outputs are written incrementally and atomically.
- [ ] Multiprocessing start method is explicit when CUDA/threads/open handles are involved.
- [ ] Process logs go through one listener or separate per-worker files.
- [ ] Nested parallelism is intentional and documented.
