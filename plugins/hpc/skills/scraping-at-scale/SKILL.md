---
name: scraping-at-scale
description: Build resumable, durable caches for high-volume web scraping/crawling on the Yale SOM HPC cluster — SQLite WAL catalogs, batched writers, single-archive body storage, and /local staging — without a GPFS metadata storm. TRIGGER when scraping or crawling many thousands of pages, building a resumable fetch catalog/cache, or storing large numbers of web artifacts on the cluster.
related:
  - acquiring-data
  - using-the-filesystem
  - parallel-python
  - accelerating-python
updated: 2026-06-10
---
# Scraping at Scale

Rule: for a large crawl, separate the **catalog** (what's cached) from the **bodies** (the data) from the **action log** (what the run did). Make the catalog durable on GPFS so the job is resumable, and never materialize a million loose files.

This is the heavy machinery for crawls of tens of thousands of pages or more. For the common case (WRDS, a few API pulls, credentials, a request-hash cache), use [acquiring data](../acquiring-data/SKILL.md) instead — most data work never needs what's here.

## Three separate stores

```text
data/raw_html/<aa>/<key>.html   # bodies, sharded by 2-char hash prefix
data/raw_json/<aa>/<key>.json
data/derived/                   # parsed outputs
data/metadata.db                # catalog: SQLite, one row per stored artifact
data/fetch_log.jsonl            # optional: JSONL, one row per fetch attempt
```

Save bodies under raw, parse separately into derived — if parsing changes, re-parse without re-fetching. Then keep two records that answer different questions:

1. **Catalog (`metadata.db`, SQLite).** One canonical row per stored artifact, keyed by `key`: `url`, `final_url`, `status`, `content_type`, `bytes`, `etag`, `last_modified`, `fetched_at`. UPSERT on each success — a 304 revalidation just updates `last_modified`/`fetched_at` without duplicate rows. Answers "what's in the cache?"
2. **Action log (`fetch_log.jsonl`, optional).** Append-only, one row per attempt: `ts`, `url`, `attempt`, `outcome` (`ok`/`cache_hit`/`retry`/`error`), `status`, `key`, `error`. Answers "what did the scraper do this run?" — including failures that produced no body.

Different shapes, different formats: the catalog has one-row-per-key identity, lookup, and updates (SQLite); the log is append-only and read as a stream (JSONL, safe to multi-write under `O_APPEND`).

**Use WAL for the catalog.** SQLite's default journal (`DELETE`) serializes readers and writers — a DuckDB query during a scrape blocks the next upsert. WAL gives concurrent reads + serialized writes, halves per-commit `fsync`, and with `synchronous = NORMAL` is durable (a crash loses at most the last in-flight transaction, never corrupts). The helper below sets it up.

## Catalog helpers

Hash the request for a stable key; shard the on-disk path by the first 2 hex chars so no directory holds more than a few thousand entries (GPFS metadata + survivable `ls`):

```python
import hashlib, json, sqlite3
from pathlib import Path

ROOT = Path("/gpfs/project/myproject/data")
CATALOG = ROOT / "metadata.db"

UPSERT_SQL = """
INSERT INTO artifacts (key, url, final_url, status, content_type, bytes,
                       etag, last_modified, fetched_at)
VALUES (:key, :url, :final_url, :status, :content_type, :bytes,
        :etag, :last_modified, :fetched_at)
ON CONFLICT(key) DO UPDATE SET
    status        = excluded.status,
    fetched_at    = excluded.fetched_at,
    etag          = excluded.etag,
    last_modified = excluded.last_modified
"""

def storage_key(url: str) -> str:
    return hashlib.sha256(url.encode()).hexdigest()                       # 64-char lowercase hex

def body_path(key: str, ext: str) -> Path:
    return ROOT / "raw_html" / key[:2] / f"{key}.{ext}"                   # data/raw_html/af/af0232....html

def write_body(path: Path, body: bytes) -> None:
    path.parent.mkdir(parents=True, exist_ok=True)
    tmp = path.with_suffix(path.suffix + ".tmp")
    tmp.write_bytes(body)
    tmp.rename(path)                                                      # atomic publish

def connect_catalog() -> sqlite3.Connection:
    conn = sqlite3.connect(CATALOG, timeout=15.0)
    conn.executescript("""
        PRAGMA synchronous   = NORMAL;     -- durable on commit; cannot corrupt on crash
        PRAGMA busy_timeout  = 15000;      -- 15 s; networked-FS lock waits can spike under contention
        PRAGMA temp_store    = MEMORY;     -- keep sort spill / temp indices off GPFS
        PRAGMA cache_size    = -65536;     -- 64 MiB page cache; trivial on a compute node
        PRAGMA foreign_keys  = ON;
    """)
    return conn

def init_catalog() -> None:
    with connect_catalog() as conn:
        conn.executescript("""
            PRAGMA journal_mode = WAL;     -- persistent in the DB file; concurrent readers, serialized writers
            CREATE TABLE IF NOT EXISTS artifacts (
                key           TEXT PRIMARY KEY,
                url           TEXT NOT NULL,
                final_url     TEXT,
                status        INTEGER NOT NULL,
                content_type  TEXT,
                bytes         INTEGER,
                etag          TEXT,
                last_modified TEXT,
                fetched_at    TEXT NOT NULL                               -- UTC ISO 8601
            );
            CREATE INDEX IF NOT EXISTS idx_artifacts_url ON artifacts(url);
        """)

class ArtifactWriter:
    """Batched UPSERT into the catalog. ALWAYS use as a context manager —
    `with` exit calls flush() so the final partial batch isn't lost."""

    def __init__(self, batch_size: int = 200):
        self.conn = connect_catalog()
        self.batch_size = batch_size
        self.pending: list[dict] = []

    def upsert(self, entry: dict) -> None:
        self.pending.append(entry)
        if len(self.pending) >= self.batch_size:
            self.flush()

    def flush(self) -> None:
        if not self.pending:
            return
        with self.conn:                                                   # one fsync per batch, not per row
            self.conn.executemany(UPSERT_SQL, self.pending)
        self.pending.clear()

    def __enter__(self):
        return self

    def __exit__(self, *exc):
        try:
            self.flush()                                                  # final flush — load-bearing
        finally:
            self.conn.close()

def append_jsonl(path: Path, entry: dict) -> None:
    line = json.dumps(entry, ensure_ascii=False) + "\n"
    with path.open("a", encoding="utf-8") as f:
        f.write(line)                                                     # < 4 KB writes are atomic under O_APPEND on POSIX
```

Usage in a fetch loop — the `with` block guarantees the final flush:

```python
with ArtifactWriter(batch_size=200) as writer:
    for url in urls:
        body, headers = fetch(url)                                        # your HTTP layer (see acquiring-data)
        key = storage_key(url)
        write_body(body_path(key, "html"), body)
        writer.upsert({
            "key": key, "url": url, "final_url": headers.get("final_url"),
            "status": headers["status"], "content_type": headers.get("content_type"),
            "bytes": len(body), "etag": headers.get("etag"),
            "last_modified": headers.get("last_modified"),
            "fetched_at": now_iso8601(),
        })
# Context exit flushed the trailing partial batch and closed the connection.
```

Trade-off: up to `batch_size` pending rows are lost on hard kill (SIGKILL, node failure) — only normal exit / exception / graceful shutdown runs `__exit__`. Pair with the SIGTERM/SIGUSR1 handler from [parallel-python](../parallel-python/SKILL.md#handle-time-limit-interruption) so a Slurm time-limit is graceful. Hard-kill is recoverable: the GPFS-resident catalog is durable up to the last batch, so the next job re-fetches only the missing keys.

Query the catalog directly, or from DuckDB:

```python
import duckdb
duckdb.sql("select status, count(*) from sqlite_scan('data/metadata.db', 'artifacts') group by status").show()
```

## GPFS operational notes

- **WAL's `-shm` wal-index is shared memory.** SQLite's [official guidance](https://sqlite.org/useovernet.html) warns WAL "does not work over a network filesystem" because `-shm` needs coherent mmap. **NFS is the documented broken case; GPFS supports coherent mmap and WAL works in practice.** Verified on Yale SOM HPC (May 2026, default_queue compute node): 4 concurrent workers UPSERTing 2000 rows each into a `/gpfs/scratch60` catalog all commit, total wall ~80 ms; on `/gpfs/home` ~130 ms. If you ever see corruption/stuck locks on a different cluster, fall back to `PRAGMA locking_mode = EXCLUSIVE` (set before first WAL access) or `PRAGMA journal_mode = DELETE`.
- **Point SQLite's tempfiles at compute-node local storage** so sorts/large indices spill off GPFS (see [using the filesystem](../using-the-filesystem/SKILL.md#compute-node-local-storage)):

```bash
workdir=$(mktemp -d "/tmp/job_${SLURM_JOB_ID:-local}.XXXXXX")             # or /local for >10 GB working sets
trap 'rm -rf "$workdir"' EXIT
export TMPDIR="$workdir"
export SQLITE_TMPDIR="$workdir"
```

## High-volume bodies — use one archive

A million-page crawl materialized as a million inodes is a [GPFS metadata burden](../using-the-filesystem/SKILL.md#metadata-warning-signs) that slows everyone's `ls`/`find`/job startup — including yours. Single-site HTML compresses well, and one `pages.zip` is far easier to `rsync`/`croc send` than 100K loose files. Append bodies to one zip with the same sharded entry path; `metadata.db` stays *outside* the archive:

```python
import zipfile

ARCHIVE = ROOT / "raw_html.zip"

def store_in_archive(key: str, body: bytes) -> None:
    arcname = f"{key[:2]}/{key}.html"                                     # af/af0232....html
    with zipfile.ZipFile(ARCHIVE, "a", compression=zipfile.ZIP_DEFLATED, compresslevel=6) as zf:
        zf.writestr(arcname, body)
```

Constraints:

- **Single writer per archive.** zip's central directory is rewritten on close; concurrent appends corrupt it. Run one writer process fed from workers via a queue (see [parallel-python](../parallel-python/SKILL.md#rung-3-bounded-queue-with-a-single-writer)), or write one archive per worker (`raw_html.<rank>.zip`) and concatenate later. (Unlike SQLite WAL, which *does* tolerate concurrent writers — don't conflate the two.)
- **Random-access reads.** `zf.read(f"{key[:2]}/{key}.html")` is O(1) via the central directory.
- **Sharing.** Ship `raw_html.zip` + `metadata.db` together; both move cleanly with `rsync`/`croc`/`rclone`.

**Stage the archive on `/local`, ship to GPFS at job end.** Appending to a zip on GPFS hammers the metadata server — every `writestr` rewrites the central directory. Build it on the compute node's [local NVMe](../using-the-filesystem/SKILL.md#compute-node-local-storage) and copy back:

```bash
# In the Slurm script:
workdir=$(mktemp -d "/local/job_${SLURM_JOB_ID:-local}.XXXXXX")
trap 'rm -rf "$workdir"' EXIT

srun .venv/bin/python src/scrape.py --archive "$workdir/raw_html.zip" &
wait $!

# Ship back on clean exit. Hard kill loses local data — design for resumability
# via the GPFS-resident metadata.db so the next job re-fetches only missing keys.
[ -f "$workdir/raw_html.zip" ] && \
    cp "$workdir/raw_html.zip" "/gpfs/project/myproject/data/raw_html.${SLURM_JOB_ID}.zip"
```

Keep `metadata.db` on GPFS so it persists across jobs. If a single job's catalog writes are throughput-bound (>~100 fetches/sec), stage it locally too and `sqlite3 src.db ".backup dst.db"` to GPFS at job end — the online backup API handles live writers.

Alternatives: **WARC** (`warcio`) when interop with crawler tooling matters; SQLite with a `bodies(key, body BLOB, meta JSON)` table when SQL over bodies + metadata helps. Avoid `tar.gz` for append — gzip-of-tar isn't cleanly appendable.

## Checklist

- [ ] Bodies sharded by 2-char hash prefix; no directory exceeds a few thousand entries.
- [ ] Each artifact has a catalog row (key, URL, status, fetched_at, etag/last_modified), UPSERTed so revalidation doesn't duplicate.
- [ ] Catalog uses WAL + `synchronous=NORMAL`; writes go through a batched `ArtifactWriter` used as a context manager (never per-row, never without a `with` to flush).
- [ ] Catalog lives on GPFS (resumable); SQLite tempfiles point at `/tmp`/`/local`.
- [ ] High-volume bodies (>~10K) go in a single archive, not loose files; long crawls build it on `/local` and ship to GPFS at job end.
- [ ] Graceful SIGTERM/SIGUSR1 handler flushes before the Slurm time limit.

## Further reading

- [SQLite UPSERT](https://www.sqlite.org/lang_upsert.html) — `ON CONFLICT … DO UPDATE`.
- [SQLite PRAGMA reference](https://sqlite.org/pragma.html) — `journal_mode`, `synchronous`, `busy_timeout`, `temp_store`, `cache_size`.
- [SQLite WAL mode](https://sqlite.org/wal.html) and [Use Over Network](https://sqlite.org/useovernet.html) — concurrency model and the networked-FS caveat.
- [DuckDB SQLite scanner](https://duckdb.org/docs/extensions/sqlite) — query SQLite catalogs from DuckDB.
- [warcio](https://github.com/webrecorder/warcio) — WARC read/write for crawler-tool interop.
