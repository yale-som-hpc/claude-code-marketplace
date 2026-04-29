---
name: using-the-filesystem
description: Use GPFS on the Yale SOM HPC cluster (/gpfs/project, /gpfs/scratch60, compute-node /tmp) without metadata storms. TRIGGER when choosing storage locations on the Yale SOM HPC cluster, moving files to/from GPFS, using cluster scratch/tmp, handling many small files on the cluster, or diagnosing GPFS I/O bottlenecks.
related:
  - overview
  - working-with-large-data
  - acquiring-data
  - installing-software
  - self-diagnosing-resource-use
updated: 2026-04-28
---
# Using the Filesystem

Rule: GPFS is good at large files and bad at metadata storms. Prefer fewer, larger files.

GPFS metadata is shared. A job that writes 100k tiny files slows down every other user's `ls`, `find`, and job startup — not just your own. This is the most common way to be unintentionally rude on the cluster.

## Where things go

```text
/gpfs/home/$USER/              personal home, not project storage
/gpfs/project/myproject/       shared project data and code
/gpfs/scratch60/$USER/         temporary shared scratch (create on first use)
/tmp                           local ephemeral temp on a compute node
```

Use `/gpfs/project/...` for shared research projects. Use `/tmp` for temporary high-I/O work inside one job. Copy final outputs back to GPFS.

`/gpfs/scratch60/$USER` does not exist by default. Create it the first time you need it:

```bash
mkdir -p /gpfs/scratch60/$USER
```

## Bad pattern: many tiny files

Bad:

```text
results/
├── result_000001.csv
├── result_000002.csv
└── ... 100,000 tiny files
```

Good:

```text
results/bootstrap_results.parquet
```

or one file per Slurm task:

```text
results/task_0001.parquet
results/task_0002.parquet
```

For append-friendly logs or scraper outputs, use compressed JSON Lines rather than thousands of files:

```python
import gzip
import json

record = {"url": url, "status": status, "text": text}
with gzip.open("/gpfs/project/myproject/data/raw/pages.jsonl.gz", "at") as f:
    f.write(json.dumps(record) + "\n")
```

For many small raw files that must stay separate, append them to a zip archive in batches:

```python
from pathlib import Path
from zipfile import ZIP_DEFLATED, ZipFile

with ZipFile("/gpfs/project/myproject/data/raw/html_pages.zip", "a", ZIP_DEFLATED) as zf:
    for path in Path("/tmp/pages").glob("*.html"):
        zf.write(path, arcname=path.name)
```

## Atomic writes

Never write directly to the final output path if a job can be killed mid-write.

```python
from pathlib import Path

output = Path("/gpfs/project/myproject/output/task_001.parquet")
tmp = output.with_suffix(output.suffix + ".tmp")

df.to_parquet(tmp)
tmp.rename(output)
```

## Local temp pattern

```bash
workdir=$(mktemp -d /tmp/${USER}_${SLURM_JOB_ID}_XXXXXX)
trap 'rm -rf "$workdir"' EXIT

cp /gpfs/project/myproject/data/input.parquet "$workdir"/
uv run python src/process.py --input "$workdir/input.parquet" --output "$workdir/output.parquet"
cp "$workdir/output.parquet" /gpfs/project/myproject/output/
```

## Moving files

Use `rsync` for repeatable transfers:

```bash
rsync -avP data/ hpc:/gpfs/project/myproject/data/raw/
```

Use compression only for compressible files:

```bash
rsync -avzP csvs/ hpc:/gpfs/project/myproject/data/raw/
rsync -avP dataset.parquet hpc:/gpfs/project/myproject/data/raw/
```

Use `croc` for one-off encrypted transfers with collaborators:

```bash
croc send large-file.zip
croc receive phrase-from-sender
```

`croc` may not be installed by default. If `which croc` fails, see [installing software](../installing-software/SKILL.md) and install it under `~/.local/bin` or `~/go/bin`.

Use `rclone` for cloud storage when configured:

```bash
rclone copy remote:bucket/path /gpfs/project/myproject/data/raw/ --progress
```

## Clean scratch

```bash
du -sh /gpfs/scratch60/$USER
find /gpfs/scratch60/$USER -mtime +30 -size +1G -ls
```

Delete old intermediates when a project is done.

## Metadata warning signs

```bash
time ls huge-directory
find output -type f | wc -l
```

If `ls` takes seconds or file counts are in the tens of thousands, consolidate.

## Checklist

- [ ] Shared work is under `/gpfs/project/...`.
- [ ] Temporary high-I/O work uses `/tmp` when possible.
- [ ] Outputs use Parquet/HDF5/zip or one file per task.
- [ ] Writes are atomic: temp file then rename.
- [ ] Scratch is cleaned regularly.
- [ ] Transfers use `rsync -avP`, not fragile one-shot copies.

## Further reading

- [rsync man page](https://www.man7.org/linux/man-pages/man1/rsync.1.html) — `-avP`, `--partial`, `--checksum`, `--exclude`.
- [croc](https://github.com/schollz/croc) — encrypted peer-to-peer transfers with collaborators.
- [rclone documentation](https://rclone.org/docs/) — cloud and S3-compatible transfers.
- [Apache Arrow / Parquet](https://arrow.apache.org/docs/) — columnar format reference (also see working-with-large-data).
