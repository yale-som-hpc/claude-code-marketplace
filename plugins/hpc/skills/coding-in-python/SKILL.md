---
name: coding-in-python
description: How to write research Python — uv environments, formatting, CLI scripts, logging. TRIGGER when authoring or editing .py files for research work. For running Python on the Yale SOM HPC cluster (sbatch, Slurm, uv on /gpfs), use running-python instead.
related:
  - programming-and-coding
  - running-python
  - parallel-python
  - accelerating-python
  - working-with-large-data
  - code-review
updated: 2026-05-22
---
# Coding in Python

Rule: locked env, portable paths, thin notebooks, runnable scripts.

For cluster execution, also load [running Python](../running-python/SKILL.md).

## Defaults

- `uv` for envs and lockfiles. Commit `pyproject.toml` and `uv.lock`; ignore `.venv/`.
- `ruff` for format + lint.
- `pathlib` for paths.
- `argparse` for scripts. Use `click` only for real subcommands.
- `logging`, not scattered `print()`, for long runs.
- `pytest` for reusable logic; smoke tests for analysis scripts.

```bash
uv init --app
uv add polars pyarrow duckdb pandas statsmodels
uv add --dev ruff pytest
uv sync --frozen
```

On HPC: sync/setup before large jobs. For long Slurm jobs call `.venv/bin/python` directly; see [running Python](../running-python/SKILL.md).

## Layout

```text
project/
├── pyproject.toml
├── uv.lock
├── README.md
├── scripts/       # entry points
├── src/           # shared helpers, or lib.py for tiny projects
├── notebooks/     # exploration
├── data/raw/      # immutable; usually ignored
├── data/derived/  # rebuildable
└── results/       # figures/tables/logs
```

## Paths

Never commit laptop/user paths: `/Users/...`, `C:\Users\...`, `/home/netid/...`, `/gpfs/scratch60/netid/...`.

```python
from pathlib import Path

REPO = Path(__file__).resolve().parents[1]
DATA = REPO / "data" / "raw" / "panel.parquet"
```

For scripts, accept paths as args:

```python
import argparse
from pathlib import Path

parser = argparse.ArgumentParser()
parser.add_argument("--input", type=Path, required=True)
parser.add_argument("--output", type=Path, required=True)
args = parser.parse_args()
```

## Style

- f-strings.
- Imports at top.
- Type hints on public/helper function signatures.
- `@dataclass(frozen=True)` for small immutable records.
- `pydantic` only for external/untrusted input.
- Context managers for every file/network/DB handle.
- Early returns to reduce nesting.
- Fixed seeds: `rng = np.random.default_rng(42)`; pass `rng` through.

## Logging

```python
import logging

log = logging.getLogger(__name__)


def main() -> None:
    logging.basicConfig(
        level=logging.INFO,
        format="%(asctime)s %(levelname)s %(name)s: %(message)s",
    )
    log.info("starting")
```

Log rows loaded, files written, model dimensions, task IDs.

## Notebooks

- Notebook for exploration/figures.
- Script for batch/rebuild/Slurm.
- Cell >30 lines: move to `src/` or `lib.py`.
- Do not import from notebooks.

## Checks

```bash
uv run ruff format .
uv run ruff check .
```

## Checklist

- [ ] Ruff clean
- [ ] No hardcoded personal paths
- [ ] Seeds set
- [ ] `uv.lock` updated if deps changed
- [ ] Small smoke test run
- [ ] HPC long jobs use `.venv/bin/python`

## Further reading

- [uv](https://docs.astral.sh/uv/)
- [ruff](https://docs.astral.sh/ruff/)
- [pathlib](https://docs.python.org/3/library/pathlib.html)
- [argparse](https://docs.python.org/3/howto/argparse.html)
