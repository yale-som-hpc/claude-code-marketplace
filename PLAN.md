# Yale SOM HPC Documentation Webapp Plan

## Goal

Build a small documentation webapp that serves the same HPC guidance to two audiences:

1. **Agents**: get clean Markdown by default, with predictable URLs and machine-readable manifests.
2. **Humans**: get a pleasant rendered website with navigation, search, examples, and cross-links.

The content should reuse the research and pages from `paulgp/hpc-usage`, but this repo becomes the durable home for Yale SOM HPC documentation and agent skills.

## Product Shape

This is not just a static marketing site. It is a **content service**.

Minimum viable behavior:

- `GET /docs/thread-control` returns Markdown by default.
- `GET /docs/thread-control?format=html` returns rendered HTML.
- Browser requests receive HTML when `Accept: text/html` is present.
- CLI/agent requests receive Markdown when no browser-like `Accept` header is present.
- `GET /docs/thread-control.md` always returns Markdown.
- `GET /docs/thread-control.html` always returns HTML.
- `GET /manifest.json` lists all docs, skills, tags, updated dates, and source paths.
- `GET /llms.txt` gives agents a compact index of available docs and how to fetch them.
- `GET /llms-full.txt` gives agents a concatenated Markdown bundle of all public docs.

## Recommended Stack

Use **Python + FastAPI + Markdown files**.

Why:

- Content negotiation is straightforward.
- Markdown-first serving is natural.
- Easy to add agent-specific endpoints (`manifest.json`, `llms.txt`, skills).
- Easy to test with `pytest` and `httpx`.
- Keeps implementation boring and inspectable.

Proposed dependencies:

- `fastapi`
- `uvicorn`
- `jinja2`
- `markdown-it-py`
- `mdit-py-plugins`
- `python-frontmatter` or a tiny TOML/YAML frontmatter parser
- `pydantic`
- `pytest`
- `httpx`
- optional later: `lunr`/`sqlite-fts5`/`tantivy` for search

Use `uv` for Python project management.

## Repository Layout

```text
.
├── pyproject.toml
├── README.md
├── PLAN.md
├── app/
│   ├── __init__.py
│   ├── main.py              # FastAPI app
│   ├── content.py           # load docs/skills from disk
│   ├── render.py            # markdown -> HTML
│   ├── negotiation.py       # markdown vs HTML response selection
│   ├── models.py            # Doc, Skill, Manifest models
│   └── templates/
│       ├── base.html
│       ├── doc.html
│       ├── index.html
│       └── skill.html
├── content/
│   ├── docs/
│   │   ├── getting-started/
│   │   ├── slurm-patterns/
│   │   ├── filesystem-patterns/
│   │   ├── work-patterns/
│   │   └── unix-practices/
│   └── skills/
│       ├── hpc-thread-control.md
│       ├── hpc-gpfs-kindness.md
│       └── hpc-gpu-efficiency.md
├── static/
│   └── style.css
├── scripts/
│   ├── import-hpc-usage.py  # copy/normalize content from paulgp/hpc-usage
│   └── check-content.py     # validate frontmatter, links, duplicate slugs
└── tests/
    ├── test_content.py
    ├── test_routes.py
    └── test_negotiation.py
```

## Content Model

Each content file is Markdown with frontmatter.

Example:

```markdown
---
title: Hidden Thread Explosions
slug: thread-control
section: work-patterns
description: Prevent NumPy, BLAS, R, and Stata from silently oversubscribing CPUs.
audience:
  - faculty
  - phd
  - ra
  - agent
tags:
  - slurm
  - threads
  - python
  - r
  - stata
updated: 2026-04-28
source: paulgp/hpc-usage#thread-control
---

# Hidden Thread Explosions

...
```

Required fields:

- `title`
- `slug`
- `section`
- `description`
- `tags`
- `updated`

Optional fields:

- `audience`
- `prerequisites`
- `related`
- `source`
- `status`: `draft | reviewed | published`

## Agent-Facing Endpoints

### Markdown docs

```text
GET /docs/{slug}
GET /docs/{slug}.md
GET /sections/{section}.md
```

### Skills

Skills are short, action-oriented Markdown files for coding agents. They should be more prescriptive than human docs.

```text
GET /skills
GET /skills/{slug}
GET /skills/{slug}.md
```

Example skill topics:

- `hpc-thread-control`
- `hpc-write-slurm-job`
- `hpc-gpfs-kindness`
- `hpc-gpu-efficiency`
- `hpc-api-caching`
- `hpc-resumable-work`
- `hpc-debug-failed-job`

### Discovery

```text
GET /manifest.json
GET /llms.txt
GET /llms-full.txt
GET /openapi.json
```

`manifest.json` should include:

```json
{
  "docs": [
    {
      "title": "Hidden Thread Explosions",
      "slug": "thread-control",
      "section": "work-patterns",
      "description": "...",
      "tags": ["slurm", "threads"],
      "markdown_url": "/docs/thread-control.md",
      "html_url": "/docs/thread-control.html",
      "updated": "2026-04-28"
    }
  ],
  "skills": []
}
```

## Human-Facing Pages

```text
GET /
GET /docs
GET /docs/{slug}.html
GET /sections/{section}
GET /skills/{slug}.html
```

Human pages should include:

- Sidebar navigation by section
- Search box
- Copy buttons for code blocks
- “For agents: Markdown version” link
- Related pages
- Last updated date
- Status badge (`draft`, `reviewed`, `published`)

## Initial Content Import

Import the current Zola docs from:

```text
/Users/klj39/src/github.com/paulgp/hpc-usage/site/content/
```

Start with the strongest pages:

1. `work-patterns/thread-control.md`
2. `getting-started/environments.md`
3. `work-patterns/bootstrap-simulations.md`
4. `work-patterns/stata.md`
5. `work-patterns/large-data.md`
6. `unix-practices/data-transfer.md`
7. `slurm-patterns/checkpointing.md`
8. `getting-started/laptop-to-cluster.md`
9. `work-patterns/interactive-tools.md`
10. `unix-practices/collaboration.md`

Then add open spike topics:

- LLM text analysis
- WRDS/API access
- Software management
- Polite cluster use
- Resumable work
- API caching/cost control
- GPFS kindness
- GPU efficiency
- Self-diagnosis guide

## Important Design Decision: Docs vs Skills

Docs explain. Skills instruct.

A doc page can be conversational and educational:

> “Why thread explosions happen, how to recognize them, and how to fix them.”

A skill should be agent-executable:

> “When writing any Slurm script, set `OMP_NUM_THREADS`, `MKL_NUM_THREADS`, and `OPENBLAS_NUM_THREADS` to `${SLURM_CPUS_PER_TASK:-1}`. Never use bare `$SLURM_CPUS_PER_TASK`.”

Each major guide should eventually have a companion skill.

## MVP Scope

### Phase 1: Content service skeleton

- Create `pyproject.toml`
- Build FastAPI app
- Load Markdown files with frontmatter
- Serve Markdown and HTML versions
- Add `/manifest.json`, `/llms.txt`, `/llms-full.txt`
- Add basic tests

### Phase 2: Import existing docs

- Write import script for Zola content
- Normalize TOML frontmatter to YAML frontmatter
- Preserve internal links where possible
- Add tags/sections/status
- Smoke test all imported pages

### Phase 3: Human UI

- Minimal HTML template
- Navigation by section
- Code block styling
- Search later; do not block MVP on search

### Phase 4: Agent skills

- Add initial skills:
  - thread control
  - Slurm job template
  - checkpointing/resumable work
  - GPFS kindness
  - GPU efficiency
  - API caching
- Add `/skills` index and manifest entries

### Phase 5: Content review

- Mark imported pages as `draft`
- Review and mark best pages as `published`
- Add cluster-specific facts carefully, with date stamps

## Non-Goals for MVP

Do not build these yet:

- User accounts
- Admin CMS
- Database
- Comments
- Analytics
- Authentication
- Complex search infrastructure
- Multi-tenant permissions

Flat files are enough.

## Deployment Options

Preferred first deployment:

```bash
uv run uvicorn app.main:app --host 0.0.0.0 --port 8000
```

Later options:

- Internal SOM VM/systemd service
- Docker/Podman container
- Fly.io/Render if public access is acceptable
- Static export fallback if needed, but content negotiation is better with a server

## Testing Strategy

Minimum tests:

- Every Markdown file has required frontmatter
- Every slug is unique
- `/docs/{slug}` returns Markdown by default
- `/docs/{slug}.html` returns HTML
- `/manifest.json` includes all docs and skills
- `/llms.txt` includes all public docs
- Internal links resolve

Smoke tests:

```bash
uv run pytest
uv run uvicorn app.main:app --reload
curl -i http://localhost:8000/docs/thread-control
curl -i -H 'Accept: text/html' http://localhost:8000/docs/thread-control
curl http://localhost:8000/manifest.json | jq .
```

## Open Questions

1. Should the site be public, Yale-only, or SOM-only?
2. Should cluster-specific facts be dated/versioned separately from general guidance?
3. Should we expose all skills publicly, or keep agent skills separate from human docs?
4. Should content be editable by PR only, or eventually via a lightweight admin UI?
5. Should we preserve the old Zola URL structure, or use simpler slug URLs?

## Recommendation

Start with the simplest useful thing: **FastAPI + Markdown files + content negotiation**.

Build the content service first. Import the best existing docs. Add agent skills second. Avoid CMS/database/search complexity until the docs are being used.
