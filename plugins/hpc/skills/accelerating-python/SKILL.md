---
name: accelerating-python
description: Speed up Python on the Yale SOM HPC cluster with profiling, DuckDB/Polars/Numba, and right-sized parallelism before reaching for multiprocessing or GPUs. TRIGGER when a Python Slurm job on the Yale SOM HPC cluster is slow/CPU-bound/memory-heavy, or when considering parallelism or GPU acceleration on the cluster.
related:
  - running-python
  - parallel-python
  - working-with-large-data
  - using-gpus
  - self-diagnosing-resource-use
updated: 2026-04-28
---
# Accelerating Python

Rule: profile first. Then choose the smallest acceleration that matches the bottleneck.

## Order of operations

1. **Measure** the slow part.
2. **Reduce data** with filters/projections before loading.
3. **Use better engines**: DuckDB, Polars, Arrow, NumPy.
4. **Vectorize** simple array/dataframe operations.
5. **Use Numba** for tight numeric loops that cannot be expressed cleanly in vectorized form.
6. **Use multiprocessing** for CPU-bound Python functions.
7. **Use GPUs** only for GPU-shaped work.

Do not start with multiprocessing if the real bottleneck is CSV parsing, GPFS metadata, a database query, or network latency.

## Quick profiler

```python
import cProfile
import pstats

with cProfile.Profile() as profiler:
    main()

stats = pstats.Stats(profiler).sort_stats("cumtime")
stats.print_stats(25)
```

For running jobs, `py-spy dump --pid PID` is often more useful if available.

## Prefer query engines for data work

DuckDB — SQL over Parquet, with predicate and projection pushdown:

```python
import duckdb

duckdb.sql("""
COPY (
  SELECT id, score
  FROM read_parquet('/gpfs/project/myproject/data/raw/*.parquet')
  WHERE score IS NOT NULL
) TO '/gpfs/project/myproject/data/derived/scores.parquet'
(FORMAT PARQUET)
""")
```

Polars — lazy expressions with a streaming sink:

```python
import polars as pl

(
    pl.scan_parquet("/gpfs/project/myproject/data/raw/*.parquet")
    .filter(pl.col("score").is_not_null())
    .select("id", "score")
    .sink_parquet("/gpfs/project/myproject/data/derived/scores.parquet")
)
```

Both push filters and projections down into the Parquet reader and stream the result — neither materializes the full table in Python. Reach for whichever idiom matches your code (SQL vs. dataframe expressions). Either is usually faster and simpler than a hand-written Python loop.

## Numba

Numba is a just-in-time compiler that translates a subset of Python and NumPy to native code via LLVM. Reach for it when you have tight loops over NumPy arrays and Python interpreter overhead dominates — not as a general "make Python fast" knob.

### Basic usage

```python
import numpy as np
from numba import njit

@njit(cache=True)
def scale(arr):
    out = np.empty_like(arr)
    for i in range(arr.shape[0]):
        out[i] = arr[i] * 2
    return out
```

Use `cache=True` so compiled code is reused across runs. `nogil=True` is redundant under `@njit` and adds noise — leave it off.

### Parallel loops

`prange` is parallel only with `parallel=True`.

```python
from numba import njit, prange

@njit(parallel=True, cache=True)
def add_one(arr):
    for i in prange(arr.shape[0]):
        arr[i] += 1
```

Parallelize the outer loop. Nested inner-loop `prange` often gives little speedup.

### Avoiding race conditions

Bad:

```python
@njit(parallel=True, cache=True)
def count_bad(counts, idx):
    for i in prange(idx.shape[0]):
        counts[idx[i]] += 1
```

Multiple workers can update the same `counts` element. Use per-worker accumulators and reduce, or use a serial loop if correctness matters more than speed. For sparse scatter-add patterns where the parallel speedup is modest, `np.add.at(counts, idx, 1)` outside Numba is thread-safe and clear.

### Compilation tax

Numba compiles on first call. Benchmark steady-state runtime separately from compile time:

```python
scale(arr)  # compile
scale(arr)  # measure this call
```

## Checklist

- [ ] The bottleneck is measured, not guessed.
- [ ] Filters/projections happen before full materialization.
- [ ] DuckDB/Polars/NumPy are considered before Python loops.
- [ ] Numba functions use numeric arrays and `@njit(cache=True)`.
- [ ] Parallel Numba avoids shared write races.
- [ ] Multiprocessing is used only when a single-process optimization is not enough.
- [ ] GPU use is justified by actual CUDA/GPU-backed code.

## Further reading

- [DuckDB documentation](https://duckdb.org/docs/) — SQL over files, Parquet pushdown, Python API.
- [Polars user guide](https://docs.pola.rs/) — lazy frames, expressions, `scan_*` / `sink_*`.
- [Numba documentation](https://numba.readthedocs.io/en/stable/) — `@njit`, `prange`, parallel reductions.
- [`cProfile` / `pstats`](https://docs.python.org/3/library/profile.html) — built-in profilers.
- [py-spy](https://github.com/benfred/py-spy) — sampling profiler for running processes (`py-spy dump --pid PID`).
- [PyTorch install matrix](https://pytorch.org/get-started/locally/) — when GPU acceleration is the right tool.
