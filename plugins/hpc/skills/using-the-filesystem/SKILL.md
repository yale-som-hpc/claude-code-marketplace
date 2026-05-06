---
name: using-the-filesystem
description: Use GPFS on the Yale SOM HPC cluster (/gpfs/project, /gpfs/scratch60, compute-node /tmp) without metadata storms. TRIGGER when choosing storage locations on the Yale SOM HPC cluster, moving files to/from GPFS, using cluster scratch/tmp, handling many small files on the cluster, or diagnosing GPFS I/O bottlenecks.
related:
  - overview
  - working-with-large-data
  - acquiring-data
  - installing-software
  - self-diagnosing-resource-use
updated: 2026-05-06
---
# Using the Filesystem

Rule: GPFS is good at large files and bad at metadata storms. Prefer fewer, larger files.

GPFS metadata is shared. A job that writes 100k tiny files slows down every other user's `ls`, `find`, and job startup — not just your own. This is the most common way to be unintentionally rude on the cluster.

## Where things go

Shared (GPFS, visible from any node):

```text
/gpfs/home/$USER/              personal home, not project storage
/gpfs/project/myproject/       shared project data and code
/gpfs/scratch60/$USER/         temporary shared scratch (create on first use)
```

Per-compute-node local (visible only on that one node, gone when the job is rescheduled):

```text
/tmp                           xfs on local NVMe, ~20 GB; default $TMPDIR; small high-I/O work
/local                         xfs on local NVMe, ~700 GB; large temp data that won't fit in /tmp
/dev/shm                       tmpfs (RAM), ~half of node memory; in-RAM ephemeral
```

Use `/gpfs/project/...` for shared research data and final outputs. For high-I/O temp data inside one job, prefer the compute-node local mounts and copy results back to GPFS at the end (see [Compute node local storage](#compute-node-local-storage) below).

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

## Compute node local storage

Each compute node has three local-storage options. Pick by size and durability:

| Mount | Type | Size | Durability | Use for |
|---|---|---|---|---|
| `/tmp` | xfs on local NVMe | ~20 GB | reboot-survives | small high-I/O working data; default `TMPDIR` |
| `/local` | xfs on local NVMe | ~700 GB | reboot-survives | large temp data that won't fit in `/tmp` |
| `/dev/shm` | tmpfs (RAM) | ~half of node RAM (≈750 GB on a 1.5 TB node) | gone on reboot, counts toward node memory | data that fits in RAM; fastest possible I/O |

Sizes verified May 2026 on a `default_queue` compute node; check yours with `df -hT /tmp /local /dev/shm`. Sub-second NVMe latency and a ~1 GB/s+ ceiling per file means a single Parquet read out of `/local` is roughly as fast as out of `/dev/shm` for sequentially-accessed files; reach for `/dev/shm` only when you need many small random reads.

**All three are shared across jobs on the same node and *not* auto-cleaned by Slurm.** Slurm sets `TMPDIR=/tmp` automatically but does *not* set `SLURM_TMPDIR`. You will see leftover files from other users' jobs. Always create a per-job subdirectory and trap-clean it; never write directly into the mount.

```bash
# Pick the storage that fits your job's working-set size:
loc=/tmp                                                                 # /local for >10 GB temp data, /dev/shm for in-RAM only

workdir=$(mktemp -d "$loc/job_${SLURM_JOB_ID:-local}.XXXXXX")
export TMPDIR="$workdir"                  # so libraries that honor TMPDIR write here too
trap 'rm -rf "$workdir"' EXIT             # clean on normal exit and on SIGTERM (when paired with `wait` below)

cp /gpfs/project/myproject/data/input.parquet "$workdir"/

# Run the work in the background and `wait` so bash can run the trap when
# Slurm sends SIGTERM at the time limit. If you call srun directly in the
# foreground, bash blocks until SIGKILL and the trap may not fire.
srun .venv/bin/python src/process.py \
    --input "$workdir/input.parquet" \
    --output "$workdir/output.parquet" &
job_pid=$!
wait "$job_pid"

cp "$workdir/output.parquet" /gpfs/project/myproject/output/
```

Notes:

- `mktemp -d "$loc/..."` with a path template works the same on Linux and macOS. Avoid `mktemp -t`; the `-t` semantics differ across systems.
- Setting `TMPDIR` makes Python's `tempfile`, R's `tempdir()`, Polars/Arrow scratch, and SQLite's spill files (also `SQLITE_TMPDIR`) write to your per-job directory.
- **Sizing.** A 20 GB Parquet intermediate is fine on `/tmp`; a 200 GB shuffle spill is not — use `/local`. If your data is genuinely smaller than node RAM and you're running random-access kernels (joblib memmaps, Faiss indices), `/dev/shm` skips the disk entirely. Remember `/dev/shm` usage counts against your job's `--mem`.
- **Why `& wait` and not just `srun ...` directly?** I tested this on the cluster (jobs 571650 and 571651). With `srun ...` (or any non-builtin) running in the foreground, bash queues the SIGTERM but cannot run the trap until the foreground command returns — by which point Slurm has already sent SIGKILL and the trap never fires. With `cmd & wait $!`, bash is parked in the `wait` builtin, which is interruptible, so the trap fires inside the `KillWait` window. SIGKILL still bypasses traps entirely; if a force-kill happens, the leftover `$workdir` will sit on the node's local disk until something else cleans it (no automatic per-job sweep) — your trap is the only reliable cleanup, so design for it to fire.

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
- [ ] Temporary high-I/O work uses a per-job subdirectory under `/tmp`, `/local`, or `/dev/shm`, picked by working-set size.
- [ ] The job creates `$workdir` with `mktemp -d` and traps `rm -rf "$workdir"` on EXIT.
- [ ] Outputs use Parquet/HDF5/zip or one file per task.
- [ ] Writes are atomic: temp file then rename.
- [ ] Scratch is cleaned regularly.
- [ ] Transfers use `rsync -avP`, not fragile one-shot copies.

## Further reading

- [rsync man page](https://www.man7.org/linux/man-pages/man1/rsync.1.html) — `-avP`, `--partial`, `--checksum`, `--exclude`.
- [croc](https://github.com/schollz/croc) — encrypted peer-to-peer transfers with collaborators.
- [rclone documentation](https://rclone.org/docs/) — cloud and S3-compatible transfers.
- [Apache Arrow / Parquet](https://arrow.apache.org/docs/) — columnar format reference (also see working-with-large-data).
