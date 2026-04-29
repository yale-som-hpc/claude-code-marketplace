---
title: Working with Large Data
slug: working-with-large-data
description: Process data larger than memory with columnar formats, query engines, pushdown, and chunked workflows.
related:
  - running-python
  - running-r
  - using-the-filesystem
  - acquiring-data
  - self-diagnosing-resource-use
updated: 2026-04-28
---

# Working with Large Data

Rule: query before loading. Store data in Parquet. Avoid loading full CSVs into memory unless you know they fit.

## Preferred formats

Use:

- Parquet for tabular data
- Arrow datasets for partitioned data
- DuckDB for SQL over files
- Polars lazy scans for Python pipelines
- HDF5 for array-like data

Avoid:

- thousands of CSV files
- repeated CSV parsing
- Excel as an intermediate format

## DuckDB example

```python
import duckdb

con = duckdb.connect()
con.execute("""
COPY (
    SELECT permno, date, ret
    FROM read_parquet('/gpfs/project/myproject/data/raw/crsp/*.parquet')
    WHERE date >= DATE '2010-01-01'
) TO '/gpfs/project/myproject/data/derived/crsp_2010_plus.parquet'
(FORMAT PARQUET)
""")
```

## Polars lazy example

```python
import polars as pl

result = (
    pl.scan_parquet("/gpfs/project/myproject/data/raw/panel/*.parquet")
    .filter(pl.col("fyear") >= 2010)
    .select(["gvkey", "fyear", "assets", "sales"])
    .group_by("fyear")
    .agg(pl.col("sales").mean())
)

result.sink_parquet("/gpfs/project/myproject/output/sales_by_year.parquet")
```

## R Arrow example

```r
library(arrow)
library(dplyr)

ds <- open_dataset("/gpfs/project/myproject/data/raw/panel")

result <- ds |>
  filter(fyear >= 2010) |>
  select(gvkey, fyear, assets, sales) |>
  group_by(fyear) |>
  summarise(mean_sales = mean(sales, na.rm = TRUE)) |>
  collect()

write_parquet(result, "/gpfs/project/myproject/output/sales_by_year.parquet")
```

## Sample first

Before a full run:

```python
sample = con.execute("""
SELECT * FROM read_parquet('/gpfs/project/myproject/data/raw/panel/*.parquet')
USING SAMPLE 10000 ROWS
""").df()
```

Use the sample to debug code, estimate memory, and check column names.

## One-file outputs

For job arrays, write one file per task:

```text
output/task_0001.parquet
output/task_0002.parquet
```

Then combine later:

```python
import polars as pl

pl.scan_parquet("output/task_*.parquet").sink_parquet("output/all_results.parquet")
```

## Domain-specific joins

Large-data tools do not make invalid joins valid. For domain datasets, use the official link tables or documented identifiers for that data source. Finance-specific patterns such as CRSP-Compustat CCM linking belong in a separate finance-data skill.

## Checklist

- [ ] Raw data is stored once in a columnar format when possible.
- [ ] Filters and column selection are pushed into the scan/query.
- [ ] Code is tested on a sample before full data.
- [ ] Outputs are Parquet/HDF5, not thousands of tiny CSVs.
- [ ] Large merges use documented identifiers or official link tables.
- [ ] Memory use is checked after the test job.
