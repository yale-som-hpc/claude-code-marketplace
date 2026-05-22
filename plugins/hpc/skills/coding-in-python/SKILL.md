---
name: coding-in-python
description: Python implementation rules for research code — uv environments, ruff, pathlib, CLI scripts, logging, seeds, portable paths, and notebook-to-script discipline. TRIGGER when editing .py files, writing Python analysis scripts, structuring a Python project, or refactoring notebook code for Yale SOM research work on a laptop or the HPC cluster.
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

Rule: use a locked environment, write portable paths, keep reusable logic in `.py` files, and make scripts runnable from a clean shell.

For cluster execution, also load [running Python](../running-python/SKILL.md). For general coding philosophy, also load [programming and coding](../programming-and-coding/SKILL.md).

## Tooling defaults

- **`uv`** for dependency management, lockfiles, and Python interpreter selection. Commit `pyproject.toml` and `uv.lock`; gitignore `.venv/`.
- **`ruff`** for lint + format. Run before committing.
- **`pathlib`** over `os.path` for filesystem work.
- **`argparse`** for batch scripts. Use `click` only when the script grows real subcommands.
- **`logging`** as the baseline. Configure once in the script entry point.
- **`pytest`** when logic is reusable or hard to inspect. For many analysis scripts, smoke tests on small real data matter more.

Optional:

- **`pyrefly`** or **`ty`** for type checking once code grows beyond a few scripts. Pick one.
- **`jupytext`** if notebooks are central and you want reviewable text diffs.

## Project setup with uv

```bash
uv init --app
uv add polars pyarrow duckdb pandas statsmodels
uv add --dev ruff pytest
uv sync --frozen
```

Commit:

```text
pyproject.toml
uv.lock
```

On the HPC, create/sync environments on the login node or in a setup job, not inside large Slurm arrays. For long Slurm jobs, call `.venv/bin/python` directly so Slurm signals reach Python; see [running Python](../running-python/SKILL.md).

## Project layout

Flat is fine for most research projects:

```text
project/
├── pyproject.toml
├── uv.lock
├── README.md
├── scripts/             # batch entry points
├── src/                 # shared helpers, or use lib.py for tiny projects
├── notebooks/           # exploratory; keep thin
├── data/raw/            # never edit; usually gitignored
├── data/derived/        # rebuildable intermediates
└── results/             # figures, tables, logs
```

Use an installable `src/<package>/` layout when shared code grows past one helper file or more than one repo imports it.

## Portable paths

Never commit laptop-specific or user-specific paths:

- `/Users/alice/Desktop/data.csv`
- `C:\Users\alice\...`
- `/home/netid/...`
- `/gpfs/scratch60/netid/...`

Use paths relative to the repo, CLI arguments, or environment variables:

```python
from pathlib import Path

REPO = Path(__file__).resolve().parents[1]
DATA = REPO / "data" / "raw" / "panel.parquet"
```

For scripts:

```python
import argparse
from pathlib import Path

parser = argparse.ArgumentParser()
parser.add_argument("--input", type=Path, required=True)
parser.add_argument("--output", type=Path, required=True)
args = parser.parse_args()
```

## Python style

- f-strings for formatting.
- Context managers (`with`) for files, database connections, and network handles.
- Early returns to reduce nesting.
- Imports at module top.
- Type hints on function signatures when they clarify intent.
- `@dataclass(frozen=True)` for small immutable records.
- `pydantic` only for validating external or untrusted inputs.
- Set seeds: `rng = np.random.default_rng(42)` and pass `rng` through; do not rely on global randomness.

## Logging pattern

```python
import logging

log = logging.getLogger(__name__)


def main() -> None:
    logging.basicConfig(
        level=logging.INFO,
        format="%(asctime)s %(levelname)s %(name)s: %(message)s",
    )
    log.info("starting analysis")
```

Use `INFO` for milestones and dimensions (`loaded 1,234,567 rows`), `WARNING` for recoverable surprises, and exceptions for invalid assumptions.

## Notebooks vs scripts

- Notebooks are for exploration, explanation, and figures.
- Scripts are for batch work, data rebuilds, and anything run by Slurm.
- A notebook cell longer than ~30 lines is a function in disguise: move it into `src/` or `lib.py` and import it.
- Never import from a notebook. Notebooks consume helpers; they do not export them.

## Smell checks

Add stricter ruff rules when code starts to sprawl:

```toml
[tool.ruff.lint]
extend-select = ["C90", "PLR"]

[tool.ruff.lint.mccabe]
max-complexity = 10

[tool.ruff.lint.pylint]
max-args = 6
max-branches = 12
max-returns = 6
max-statements = 50
```

Ad hoc audits:

```bash
uvx radon cc -s -a scripts src
uvx vulture scripts src
```

## Checklist

- [ ] `uv run ruff format . && uv run ruff check .` clean
- [ ] No hardcoded laptop, home, scratch, or user-specific paths
- [ ] Seeds set on stochastic code paths
- [ ] `uv.lock` committed when dependencies changed
- [ ] Small realistic smoke test run
- [ ] On HPC, long Slurm jobs call `.venv/bin/python` directly

## Further reading

- [uv documentation](https://docs.astral.sh/uv/)
- [ruff documentation](https://docs.astral.sh/ruff/)
- [Python pathlib](https://docs.python.org/3/library/pathlib.html)
- [argparse tutorial](https://docs.python.org/3/howto/argparse.html)
- [pytest documentation](https://docs.pytest.org/)
- [Ruff rule index](https://docs.astral.sh/ruff/rules/)
