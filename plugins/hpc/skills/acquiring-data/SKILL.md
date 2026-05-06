---
name: acquiring-data
description: Download, query, scrape, and call APIs from the Yale SOM HPC cluster without leaking credentials, repeating expensive requests, or getting the cluster's outbound IP blocked. TRIGGER when fetching datasets onto /gpfs, calling WRDS/REST APIs from the Yale SOM HPC cluster, scraping from the cluster, caching downloads on the cluster, or handling credentials on the cluster.
related:
  - using-the-filesystem
  - working-with-large-data
  - running-python
  - installing-software
  - self-diagnosing-resource-use
updated: 2026-05-06
---
# Acquiring Data

Rule: fetch once, cache raw responses, parse separately, and never put credentials in scripts.

This skill covers WRDS, REST APIs, web scraping, paid LLM APIs, direct downloads, cloud storage, and collaborator handoffs.

## Credentials

Bad:

```python
password = "my-wrds-password"
api_key = "sk-..."
```

Good:

```bash
chmod 600 ~/.pgpass ~/.env 2>/dev/null || true
```

```python
import os

api_key = os.environ["MY_API_KEY"]
```

For project jobs, load secrets from a protected `.env` file or user environment. Do not commit `.env`, put it in `.gitignore`.

## Direct downloads

Prefer downloading directly on the cluster when allowed:

```bash
wget -c -O /gpfs/project/myproject/data/raw/file.zip "https://example.com/file.zip"
curl -L --retry 5 --retry-delay 10 -o file.zip "https://example.com/file.zip"
```

Use `rsync`/`croc` for collaborator files; see [using the filesystem](../using-the-filesystem/SKILL.md).

## WRDS pattern

Download once to project storage, then analyze local extracts.

```python
import wrds

conn = wrds.Connection()
query = """
select permno, date, ret
from crsp.msf
where date >= '2010-01-01'
"""
df = conn.raw_sql(query, date_cols=["date"])
df.to_parquet("/gpfs/project/myproject/data/raw/crsp_msf_2010_plus.parquet")
```

Do not run the same WRDS extract repeatedly.

## Postgres / WRDS connections from parallel workers

For direct Postgres access (including WRDS, which is Postgres under the hood), keep credentials out of code with a `pg_service.conf` file in `$HOME` and reference connections by service name:

```ini
# ~/.pg_service.conf — chmod 600
[wrds]
host=wrds-pgdata.wharton.upenn.edu
port=9737
dbname=wrds
user=yourwrdsid

[mydb]
host=db.example.com
dbname=research
user=yournetid
```

Combined with `~/.pgpass` (already `chmod 600`), code stays free of secrets:

```python
import psycopg

with psycopg.connect("service=wrds") as conn, conn.cursor() as cur:
    cur.execute("select permno, date, ret from crsp.msf where date >= %s", ("2010-01-01",))
    rows = cur.fetchall()
```

When parallel workers share a database, **always use a connection pool**. Do not let each worker open its own short-lived connection — Postgres servers cap concurrent connections, WRDS especially, and naive parallelism will get you rate-limited or blocked:

```python
from psycopg_pool import ConnectionPool

# One pool per process. With multiprocessing, create the pool inside the worker,
# not in the parent — connections cannot survive a fork.
pool = ConnectionPool("service=wrds", min_size=2, max_size=8)

def fetch(permno: int):
    with pool.connection() as conn, conn.cursor() as cur:
        cur.execute("select date, ret from crsp.msf where permno = %s", (permno,))
        return cur.fetchall()
```

Bound `max_size` deliberately. A pool of 8 across 4 worker processes means 32 concurrent connections — past most Postgres limits. Set the pool size to a small number per worker (2–4) and let the pool queue further requests.

## Request-hash cache

Use this for paid APIs, web pages, embeddings, LLM calls, and slow endpoints.

```python
import hashlib
import json
from pathlib import Path

CACHE_DIR = Path("/gpfs/project/myproject/cache/api")
CACHE_DIR.mkdir(parents=True, exist_ok=True)

def cache_key(payload: dict) -> str:
    encoded = json.dumps(payload, sort_keys=True, ensure_ascii=False).encode()
    return hashlib.sha256(encoded).hexdigest()

def cached_call(payload: dict):
    path = CACHE_DIR / f"{cache_key(payload)}.json"
    if path.exists():
        return json.loads(path.read_text())

    response = call_expensive_api(payload)

    tmp = path.with_suffix(".json.tmp")
    tmp.write_text(json.dumps(response))
    tmp.rename(path)
    return response
```

## Rate limits and retries

With `tenacity`:

```python
from tenacity import retry, wait_exponential, stop_after_attempt

@retry(wait=wait_exponential(min=1, max=60), stop=stop_after_attempt(6))
def fetch(url: str):
    response = session.get(url, timeout=30)
    response.raise_for_status()
    return response
```

Also add a deliberate delay when scraping:

```python
import time

time.sleep(1.0)
```

No dependency version:

```python
import time
import requests

session = requests.Session()

def fetch(url: str, attempts: int = 5):
    for attempt in range(attempts):
        response = session.get(url, timeout=30)
        if response.status_code == 429 and attempt < attempts - 1:
            retry_after = int(response.headers.get("Retry-After", "0") or "0")
            time.sleep(retry_after or 2 ** attempt)
            continue
        try:
            response.raise_for_status()
            return response
        except requests.HTTPError:
            if attempt == attempts - 1:
                raise
            time.sleep(2 ** attempt)
```

All cluster jobs may share one outbound IP. One user's aggressive scraper can get everyone blocked.

## Store raw + metadata

```text
data/raw_html/<aa>/<key>.html   # bodies, sharded by 2-char hash prefix
data/raw_json/<aa>/<key>.json
data/derived/                   # parsed outputs
data/metadata.db                # catalog: SQLite, one row per stored artifact
data/fetch_log.jsonl            # optional: JSONL, one row per fetch attempt
```

Save bodies under raw, parse separately into derived. If parsing changes, re-parse without re-fetching.

Keep two things separate — they answer different questions and want different storage:

1. **Catalog (`metadata.db`, SQLite).** One canonical row per stored artifact, keyed by `key`. Columns: `key`, `url`, `final_url`, `status`, `content_type`, `bytes`, `etag`, `last_modified`, `fetched_at`. Indexed by `key` (primary key) and `url`. UPSERT on each successful fetch — a 304 revalidation just updates `last_modified` and `fetched_at` without piling up duplicate rows. Answers "what is this artifact, and what's in the cache?"
2. **Action log (`fetch_log.jsonl`, optional).** Append-only, one row per attempt: `ts`, `url`, `attempt`, `outcome` (`ok` / `cache_hit` / `retry` / `error`), `status`, `key` (if a body was stored), `error` (if any). Answers "what did the scraper do during this run?" — including failures and retries that produced no body.

Different shapes, different formats. The catalog has natural one-row-per-key identity, lookup-by-key, and updates — SQLite expresses that. The log is append-only, time-ordered, and only ever read as a stream — JSONL is the simpler fit and is safe to multi-write under O_APPEND.

**Use WAL for the catalog.** SQLite's default journal mode (`DELETE`, the rollback journal) serializes readers and writers — a DuckDB query during a scrape would block the next `upsert_artifact`, and vice versa. WAL gives you concurrent reads + serialized writes, halves the per-commit `fsync` count, and with `synchronous = NORMAL` is fully durable (a power cut loses at most the last in-flight transaction, never corrupts the file). For any catalog you query while scraping or scrape from multiple workers, WAL is the right journal mode; the `connect_catalog()` helper below sets it up, and the operational notes after confirm it works on Yale SOM HPC GPFS.

Hash the request for a stable key; shard the on-disk path by the first 2 hex chars so no single directory holds more than a few thousand entries (matters for GPFS metadata and survivable `ls`):

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
        PRAGMA synchronous   = NORMAL;     -- durable on commit; cannot corrupt on crash. FULL is overkill for a research catalog
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

Usage in a fetch loop — the `with` block is what guarantees the final flush:

```python
with ArtifactWriter(batch_size=200) as writer:
    for url in urls:
        body, headers = fetch(url)                                        # your HTTP layer
        key = storage_key(url)
        write_body(body_path(key, "html"), body)
        writer.upsert({
            "key": key, "url": url, "final_url": headers.get("final_url"),
            "status": headers["status"], "content_type": headers.get("content_type"),
            "bytes": len(body), "etag": headers.get("etag"),
            "last_modified": headers.get("last_modified"),
            "fetched_at": now_iso8601(),
        })
        # optional: append_jsonl(ROOT / "fetch_log.jsonl",
        #                       {"ts": now_iso8601(), "url": url, "outcome": "ok", "key": key})
# Context exit flushed the trailing partial batch and closed the connection.
```

Trade-off: pending rows (up to `batch_size`) are lost on hard kill (SIGKILL, node failure) — only on normal exit, exception, or graceful shutdown does `__exit__` run. Pair with the SIGTERM/SIGUSR1 handler from [parallel-python](../parallel-python/SKILL.md#handle-time-limit-interruption) so a Slurm time-limit is graceful (handler sets a flag, the loop exits, `with` flushes). Hard-kill is recoverable: the catalog on GPFS is durable up to the last batch, so the next job re-fetches only the keys missing from it. `batch_size=200` typically commits within a second or two of fetching — tune up for fast APIs, down if losing the last batch is expensive.

Query the catalog directly, or from DuckDB with `sqlite_scan`:

```python
import duckdb
duckdb.sql("select status, count(*) from sqlite_scan('data/metadata.db', 'artifacts') group by status").show()
```

Concurrent workers each open their own `ArtifactWriter` — WAL serializes the underlying writes via file locking and readers don't block. **Different from the zip**: the zip's central directory genuinely can't tolerate concurrent appends (hence the queue+writer pattern below); SQLite's WAL can. Don't conflate the two single-writer stories.

Two GPFS-specific operational notes (reassurance, not workarounds):

- **WAL's `-shm` wal-index is shared memory.** SQLite's [official guidance](https://sqlite.org/useovernet.html) warns that WAL "does not work over a network filesystem" because `-shm` requires coherent mmap. NFS is the documented broken case; GPFS supports coherent mmap and WAL works in practice. Verified on Yale SOM HPC (May 2026, default_queue compute node): 4 concurrent worker processes UPSERTing 2000 rows each into a `/gpfs/scratch60` SQLite catalog with these pragmas all commit successfully, total wall ~80 ms; on `/gpfs/home` ~130 ms. If you ever observe corruption or stuck locks on a different cluster, fall back to `PRAGMA locking_mode = EXCLUSIVE` (single-opener, no `-shm` needed — set *before* the first WAL access) or `PRAGMA journal_mode = DELETE` (rollback journal, slower writes, works on any POSIX FS).
- **Point SQLite's tempfiles at compute-node local storage.** Sorts and large indices spill through `SQLITE_TMPDIR`; keep them off GPFS by setting it to the per-job `$workdir` you create on `/tmp` or `/local` (see [using the filesystem](../using-the-filesystem/SKILL.md#compute-node-local-storage)):

```bash
workdir=$(mktemp -d "/tmp/job_${SLURM_JOB_ID:-local}.XXXXXX")             # or /local for >10 GB working sets
trap 'rm -rf "$workdir"' EXIT
export TMPDIR="$workdir"
export SQLITE_TMPDIR="$workdir"
```

### High-volume crawls — use one archive

A million-page crawl materialized as a million inodes is a [GPFS metadata burden](../using-the-filesystem/SKILL.md#metadata-warning-signs) that slows everyone's `ls`, `find`, and job startup — including yours. Single-site crawls also have heavy HTML repetition that compresses well, and a single `pages.zip` is far easier to `rsync` or `croc send` to a collaborator than 100K loose files. Append bodies to one zip, keeping the same sharded entry path; `metadata.jsonl` stays *outside* the archive:

```python
import zipfile

ARCHIVE = ROOT / "raw_html.zip"

def store_in_archive(key: str, body: bytes) -> None:
    arcname = f"{key[:2]}/{key}.html"                                     # af/af0232....html
    with zipfile.ZipFile(ARCHIVE, "a", compression=zipfile.ZIP_DEFLATED, compresslevel=6) as zf:
        zf.writestr(arcname, body)
```

Constraints:

- **Single writer per archive.** zip's central directory is rewritten on close; concurrent appends from multiple processes corrupt the archive. Either run one writer process fed from workers via a queue (see [parallel-python](../parallel-python/SKILL.md#rung-3-bounded-queue-with-a-single-writer)), or write one archive per worker (`raw_html.<rank>.zip`) and concatenate later.
- **Random-access reads.** `zf.read(f"{key[:2]}/{key}.html")` is O(1) via the central directory.
- **Sharing.** Ship `raw_html.zip` + `metadata.db` together; both are single files that move cleanly with `rsync`, `croc send`, or `rclone`.

**Stage the archive on `/local`, ship to GPFS at job end.** A long-running fetch loop appending to a zip on GPFS hammers the metadata server — every `writestr` rewrites the central directory, so 1M fetches is 1M metadata round-trips. Build the archive on the compute node's [local NVMe](../using-the-filesystem/SKILL.md#compute-node-local-storage) (~700 GB on `/local`) and copy the finished file back:

```bash
# In the Slurm script:
workdir=$(mktemp -d "/local/job_${SLURM_JOB_ID:-local}.XXXXXX")
trap 'rm -rf "$workdir"' EXIT

srun .venv/bin/python src/scrape.py --archive "$workdir/raw_html.zip" &
wait $!

# Ship back on clean exit. Hard kill (SIGKILL, node failure) loses local data —
# design for resumability via the GPFS-resident metadata.db catalog so the next
# job can re-fetch only the keys missing from the archive.
[ -f "$workdir/raw_html.zip" ] && \
    cp "$workdir/raw_html.zip" "/gpfs/project/myproject/data/raw_html.${SLURM_JOB_ID}.zip"
```

The catalog (`metadata.db`) stays on GPFS so it persists across jobs and the next scrape knows what's already cached. If a single job's catalog writes are throughput-bound (cached-response crawls, fast APIs > ~100 fetches/sec), stage it locally too and `sqlite3 src.db ".backup dst.db"` to GPFS at job end — the online backup API handles live writers without taking the DB down.

Alternatives: WARC (the standard web-archive format; `warcio`) when interop with crawler tooling matters; SQLite with a `bodies(key, body BLOB, meta JSON)` table when SQL over bodies + metadata is useful. Avoid `tar.gz` for append — gzip-of-tar isn't cleanly appendable.

## Cost cap

```python
MAX_BUDGET_DOLLARS = 50.0
spent = 0.0

for request in requests:
    if spent >= MAX_BUDGET_DOLLARS:
        raise RuntimeError(f"budget exceeded: ${spent:.2f}")
    result = cached_call(request)
    spent += estimate_cost(result)
```

## Checklist

- [ ] Credentials are outside scripts and not committed.
- [ ] Raw downloaded/scraped/API data is saved before parsing.
- [ ] Expensive or repeated requests are cached by hash.
- [ ] Body paths are sharded by 2-char hash prefix so no directory exceeds a few thousand entries.
- [ ] Each stored artifact has a row in the `metadata.db` catalog (key, URL, status, fetched_at, etag/last_modified), upserted so revalidation doesn't duplicate rows.
- [ ] Catalog writes go through a batched `ArtifactWriter` used as a context manager — never per-row UPSERTs in a loop, never a loop without a `with` block to flush the trailing batch.
- [ ] (Optional) Fetch attempts are logged to `fetch_log.jsonl` so failed runs can be reconstructed.
- [ ] High-volume crawls (>~10K artifacts) store bodies in a single archive, not as individual files (GPFS metadata + collaborator sharing).
- [ ] Long-running scrapes build the archive on compute-node `/local` (NVMe) and ship to GPFS at job end, not append-on-GPFS in a loop.
- [ ] Retries use exponential backoff.
- [ ] Scrapers sleep and respect robots/rate limits.
- [ ] Paid API jobs estimate cost before full run.
- [ ] Shared project cache prevents multiple RAs from paying for the same call.

## Further reading

- [WRDS support pages](https://wrds-www.wharton.upenn.edu/pages/support/) — `wrds` Python package, `.pgpass`, schemas.
- [psycopg 3](https://www.psycopg.org/psycopg3/docs/) — connections, cursors, type adapters.
- [psycopg_pool](https://www.psycopg.org/psycopg3/docs/advanced/pool.html) — `ConnectionPool` sizing, `min_size`/`max_size`, lifecycle.
- [PostgreSQL service file](https://www.postgresql.org/docs/current/libpq-pgservice.html) — `pg_service.conf` indirection so code references `service=wrds` instead of credentials.
- [httpx](https://www.python-httpx.org/) — modern sync/async HTTP client (preferred over `requests` for new code).
- [tenacity](https://tenacity.readthedocs.io/en/latest/) — retry decorators, backoff, stop conditions.
- [RFC 9309 (robots.txt)](https://www.rfc-editor.org/rfc/rfc9309.html) — the spec scrapers should respect.
- [`hashlib`](https://docs.python.org/3/library/hashlib.html) — request-hash cache keys.
- [SQLite UPSERT](https://www.sqlite.org/lang_upsert.html) — `ON CONFLICT … DO UPDATE` for revalidation-friendly catalogs.
- [SQLite PRAGMA reference](https://sqlite.org/pragma.html) — `journal_mode`, `synchronous`, `busy_timeout`, `temp_store`, `cache_size`.
- [SQLite WAL mode](https://sqlite.org/wal.html) and [Use Over Network](https://sqlite.org/useovernet.html) — concurrency model and the networked-FS caveat.
- [DuckDB SQLite scanner](https://duckdb.org/docs/extensions/sqlite) — `sqlite_scan('file.db', 'table')` to query SQLite catalogs from DuckDB.
