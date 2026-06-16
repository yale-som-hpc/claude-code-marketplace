---
name: acquiring-data
description: Download, query, scrape, and call APIs from the Yale SOM HPC cluster without leaking credentials, repeating expensive requests, or getting the shared outbound IP blocked. TRIGGER when fetching datasets onto /gpfs, calling WRDS/REST APIs, scraping, caching downloads, or handling credentials on the cluster.
related:
  - using-the-filesystem
  - accelerating-python
  - running-python
  - scraping-at-scale
  - installing-software
  - self-diagnosing-resource-use
updated: 2026-06-10
---
# Acquiring Data

Rule: fetch once, cache raw responses, parse separately, and never put credentials in scripts.

This skill covers WRDS, REST APIs, web scraping, paid LLM APIs, direct downloads, and collaborator handoffs. For a *high-volume* crawl (tens of thousands of pages or more), see [scraping at scale](../scraping-at-scale/SKILL.md).

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

For project jobs, load secrets from a protected `.env` file or user environment. Do not commit `.env`; put it in `.gitignore`.

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

Bound `max_size` deliberately. A pool of 8 across 4 worker processes means 32 concurrent connections — past most Postgres limits. Set 2–4 per worker and let the pool queue further requests.

## Request-hash cache

Use this for paid APIs, web pages, embeddings, LLM calls, and slow endpoints.

```python
import hashlib
import json
import os
import tempfile
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

    fd, tmp_name = tempfile.mkstemp(prefix=path.name, suffix=".tmp", dir=path.parent)
    with os.fdopen(fd, "w") as f:
        json.dump(response, f)
    os.replace(tmp_name, path)
    return response
```

For highly parallel jobs, avoid many workers discovering the same missing key at once. Precompute the shared cache in one job, or guard writes with a lock, so duplicates don't all pay for the same request.

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

Add a deliberate delay when scraping, and respect `Retry-After`:

```python
import time
time.sleep(1.0)
```

**All cluster jobs may share one outbound IP. One user's aggressive scraper can get everyone blocked** — throttle, cache, and respect `robots.txt`.

## Store raw, parse separately

Save raw responses under `data/raw/`, parse separately into `data/derived/`. If parsing changes, re-parse without re-fetching. Write bodies atomically (temp + rename), and shard on-disk paths by a 2-char hash prefix so no directory holds more than a few thousand files (GPFS metadata health, survivable `ls`):

```text
data/raw_html/<aa>/<key>.html   # bodies, sharded by 2-char hash prefix
data/derived/                   # parsed outputs
data/metadata.db                # catalog: SQLite, one row per stored artifact
```

Keep a small SQLite catalog (`metadata.db`) with one row per artifact (`key`, `url`, `status`, `bytes`, `etag`, `last_modified`, `fetched_at`), UPSERTed so revalidation doesn't duplicate rows — it tells the next run what's already cached.

For a **high-volume crawl** — durable batched catalogs, WAL-on-GPFS, single-archive body storage, `/local` staging — see [scraping at scale](../scraping-at-scale/SKILL.md). Don't materialize a million loose files on GPFS.

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
- [ ] Parallel DB access uses a bounded connection pool, not per-worker connections.
- [ ] Retries use exponential backoff; scrapers sleep and respect robots/rate limits.
- [ ] Paid API jobs estimate cost before a full run.
- [ ] A shared project cache prevents multiple RAs from paying for the same call.
- [ ] High-volume crawls follow [scraping at scale](../scraping-at-scale/SKILL.md) (catalog + archive, not loose files).

## Further reading

- [WRDS support pages](https://wrds-www.wharton.upenn.edu/pages/support/) — `wrds` Python package, `.pgpass`, schemas.
- [psycopg 3](https://www.psycopg.org/psycopg3/docs/) and [psycopg_pool](https://www.psycopg.org/psycopg3/docs/advanced/pool.html) — connections, cursors, `ConnectionPool` sizing.
- [PostgreSQL service file](https://www.postgresql.org/docs/current/libpq-pgservice.html) — `pg_service.conf` so code references `service=wrds` instead of credentials.
- [httpx](https://www.python-httpx.org/) — modern sync/async HTTP client (preferred over `requests` for new code).
- [tenacity](https://tenacity.readthedocs.io/en/latest/) — retry decorators, backoff, stop conditions.
- [RFC 9309 (robots.txt)](https://www.rfc-editor.org/rfc/rfc9309.html) — the spec scrapers should respect.
- [`hashlib`](https://docs.python.org/3/library/hashlib.html) — request-hash cache keys.
