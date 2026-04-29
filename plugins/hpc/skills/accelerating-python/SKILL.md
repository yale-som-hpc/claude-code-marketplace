---
name: accelerating-python
description: Speed up Python by choosing better algorithms, vectorization, DuckDB/Polars, Numba, or GPUs before adding blind multiprocessing. TRIGGER when Python jobs are slow, CPU-bound, memory-heavy, or considering parallelism/GPU acceleration.
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

This is often faster and simpler than writing Python loops.

## Numba for tight numeric loops

Use Numba when you have loops over NumPy arrays and Python overhead dominates.

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

Use `cache=True` so compiled code is reused across runs.

## Parallel Numba

`prange` is parallel only with `parallel=True`.

```python
from numba import njit, prange

@njit(parallel=True, cache=True)
def add_one(arr):
    for i in prange(arr.shape[0]):
        arr[i] += 1
```

Parallelize the outer loop. Nested inner-loop `prange` often gives little speedup.

## Avoid race conditions

Bad:

```python
@njit(parallel=True, cache=True)
def count_bad(counts, idx):
    for i in prange(idx.shape[0]):
        counts[idx[i]] += 1
```

Multiple workers can update the same `counts` element. Use per-worker accumulators and reduce, or use a serial loop if correctness matters more than speed.

## Compilation tax

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
