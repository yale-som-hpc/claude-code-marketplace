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
