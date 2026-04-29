---
name: acquiring-data
description: Download, query, scrape, and call APIs from the Yale SOM HPC cluster without leaking credentials, repeating expensive requests, or getting the cluster's outbound IP blocked. TRIGGER when fetching datasets onto /gpfs, calling WRDS/REST APIs from the Yale SOM HPC cluster, scraping from the cluster, caching downloads on the cluster, or handling credentials on the cluster.
related:
  - using-the-filesystem
  - working-with-large-data
  - running-python
  - installing-software
  - self-diagnosing-resource-use
updated: 2026-04-28
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

Do not run the same WRDS extract inside every analysis job.

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

All cluster jobs may share one outbound IP. One user's aggressive scraper can get everyone blocked.

## Store raw before parsing

```text
data/raw_html/
data/raw_json/
data/derived/
```

If parsing changes, re-parse raw responses without re-fetching.

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
