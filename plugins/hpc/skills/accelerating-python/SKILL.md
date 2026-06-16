---
name: accelerating-python
description: Make slow or memory-bound Python fast on the Yale SOM HPC cluster — profile, push work into DuckDB/Polars/Arrow, vectorize/Numba, then parallelize or use a GPU. Also covers data larger than a job's RAM (Parquet, query engines, sampling). TRIGGER when a Python job on the cluster is slow, CPU-bound, or memory-heavy, when a dataset on /gpfs does not fit a job's RAM, or when weighing parallelism or GPU acceleration.
related:
  - running-python
  - parallel-python
  - using-the-filesystem
  - using-gpus
  - self-diagnosing-resource-use
updated: 2026-06-10
---
# Accelerating Python

Rule: profile first; query before loading; then pick the smallest acceleration that matches the bottleneck. Most "slow Python" and "data too big" problems are solved before you reach for multiprocessing or a GPU.

## Order of operations

1. **Measure** the slow part — don't guess.
2. **Reduce data** with filters/projections *before* loading (push them into the file reader).
3. **Use better engines**: DuckDB, Polars, Arrow, NumPy — they stream and push down.
4. **Vectorize** simple array/dataframe operations.
5. **Numba** for tight numeric loops that don't vectorize cleanly.
6. **Multiprocessing** for CPU-bound Python — see [parallel python](../parallel-python/SKILL.md).
7. **GPU** only for GPU-shaped work — see [using GPUs](../using-gpus/SKILL.md).

Do not start at step 6 if the real bottleneck is CSV parsing, GPFS metadata, a database query, or network latency.

## Quick profiler

```python
import cProfile
import pstats

with cProfile.Profile() as profiler:
    main()

stats = pstats.Stats(profiler).sort_stats("cumtime")
stats.print_stats(25)
```

For a running job, `py-spy dump --pid PID` is often more useful (if installed).

## Store data in a columnar format

- **Parquet** for tabular data, **Arrow** datasets for partitioned data, **HDF5** for array-like data.
- Avoid thousands of CSVs, repeated CSV parsing, and Excel as an intermediate format.

One reused Parquet beats 10k CSVs for both speed and GPFS metadata health.

## Query before loading

The single biggest win for both speed and memory: filter and project *in the reader* so the full table never enters Python. DuckDB (SQL) and Polars (lazy) both push predicates/projections into the Parquet scan and stream the result — pick whichever idiom matches your code.

DuckDB — SQL over Parquet with pushdown:

```python
import duckdb

duckdb.sql("""
COPY (
  SELECT gvkey, fyear, sales
  FROM read_parquet('/gpfs/project/myproject/data/raw/panel/*.parquet')
  WHERE fyear >= 2010
) TO '/gpfs/project/myproject/data/derived/sales.parquet'
(FORMAT PARQUET)
""")
```

Polars — lazy expressions with a streaming sink:

```python
import polars as pl

(
    pl.scan_parquet("/gpfs/project/myproject/data/raw/panel/*.parquet")
    .filter(pl.col("fyear") >= 2010)
    .select("gvkey", "fyear", "sales")
    .sink_parquet("/gpfs/project/myproject/data/derived/sales.parquet")
)
```

Neither materializes the full table in Python. Either is usually faster and simpler than a hand-written Python loop.

## Installing the CLI tools

Neither `duckdb` nor `qsv` is module-loadable on the cluster (`module spider duckdb` / `qsv` both return "Unable to find"). Install them as static binaries in `~/.local/bin` — see [installing software](../installing-software/SKILL.md#user-binaries) for PATH setup:

```bash
mkdir -p ~/.local/bin
# DuckDB CLI — single static binary, latest release
curl -L https://github.com/duckdb/duckdb/releases/latest/download/duckdb_cli-linux-amd64.zip -o /tmp/duckdb.zip
unzip -o /tmp/duckdb.zip -d ~/.local/bin && rm /tmp/duckdb.zip
duckdb --version
# qsv — pick the musl prebuild (x86_64-unknown-linux-musl) from
# https://github.com/jqnatividad/qsv/releases ; same drop-in install.
```

Inside a Python project you also get DuckDB via `uv add duckdb` — that's the *library* (`import duckdb`); the command-line `duckdb` is a separate binary, both backed by the same engine. CLI for ad-hoc inspection, library inside scripts.

## Quick command-line inspection

Look before you script:

```bash
duckdb -c "DESCRIBE SELECT * FROM 'data.csv'"
duckdb -c "SELECT category, COUNT(*) FROM 'data.csv' GROUP BY category ORDER BY 2 DESC"
duckdb -c "COPY (SELECT * FROM 'data.csv') TO 'data.parquet' (FORMAT PARQUET)"   # convert reusable extracts
qsv stats data.csv          # fast CSV triage
```

## Sample first

Before a full run, debug code and estimate memory on a sample:

```python
sample = duckdb.sql("""
SELECT * FROM read_parquet('/gpfs/project/myproject/data/raw/panel/*.parquet')
USING SAMPLE 10000 ROWS
""").df()
```

## Numba

A JIT compiler for a subset of Python/NumPy. Reach for it when tight loops over NumPy arrays dominate and interpreter overhead is the bottleneck — not as a general "make Python fast" knob.

```python
import numpy as np
from numba import njit

@njit(cache=True)            # cache=True reuses compiled code across runs
def scale(arr):
    out = np.empty_like(arr)
    for i in range(arr.shape[0]):
        out[i] = arr[i] * 2
    return out
```

Parallel loops need `parallel=True`; parallelize the **outer** loop:

```python
from numba import njit, prange

@njit(parallel=True, cache=True)
def add_one(arr):
    for i in prange(arr.shape[0]):
        arr[i] += 1
```

**Race warning:** parallel `prange` writing to a shared array (e.g. `counts[idx[i]] += 1`) races. Use per-worker accumulators and reduce, or `np.add.at(counts, idx, 1)` outside Numba (thread-safe). And Numba compiles on first call — benchmark the *second* call to exclude compile time.

## SQLite for small caches

Local caches and lookup tables; enable WAL when multiple readers use the cache:

```python
import sqlite3
conn = sqlite3.connect("/gpfs/project/myproject/cache/lookup.db")
conn.execute("PRAGMA journal_mode=WAL")
```

One connection per thread/process; SQLite serializes writes — keep one writer path.

## One file per array task

For job arrays, write `output/task_0001.parquet`, …, then combine — don't mutate one big Parquet. The full append/atomic-write patterns live in [using the filesystem](../using-the-filesystem/SKILL.md).

```python
pl.scan_parquet("output/task_*.parquet").sink_parquet("output/all_results.parquet")
```

## Other languages

Writing R? `arrow` + `dplyr` over `open_dataset()` gives the same pushdown — see [coding in R](../coding-in-r/SKILL.md) and [running R](../running-r/SKILL.md).

## Joins need real keys

Large-data tools do not make invalid joins valid. Use documented identifiers or official link tables (e.g. CRSP-Compustat CCM linking); finance-specific linking belongs in a finance-data skill.

## Checklist

- [ ] The bottleneck is measured, not guessed.
- [ ] Filters/projections are pushed into the scan/query before materializing.
- [ ] Raw data is stored once in Parquet; files inspected with DuckDB/qsv before big scripts.
- [ ] Code is tested on a sample before full data; memory checked after the test job.
- [ ] DuckDB/Polars/NumPy are tried before Python loops; Numba uses `@njit(cache=True)` and avoids shared-write races.
- [ ] Multiprocessing/GPU are used only when single-process optimization isn't enough.
- [ ] Outputs are Parquet/HDF5, not thousands of tiny CSVs; large merges use documented keys.

## Further reading

- [DuckDB documentation](https://duckdb.org/docs/) — SQL over Parquet/CSV/Arrow, `COPY ... TO`, sampling, pushdown.
- [Polars user guide](https://docs.pola.rs/) — lazy frames, `scan_*` / `sink_*`, expressions.
- [Apache Arrow](https://arrow.apache.org/docs/) — Parquet, Arrow datasets, columnar concepts.
- [Numba documentation](https://numba.readthedocs.io/en/stable/) — `@njit`, `prange`, parallel reductions.
- [`cProfile` / `pstats`](https://docs.python.org/3/library/profile.html) and [py-spy](https://github.com/benfred/py-spy) — profilers.
- [qsv](https://github.com/jqnatividad/qsv) / [h5py](https://docs.h5py.org/en/stable/) — CSV triage; HDF5 from Python.
