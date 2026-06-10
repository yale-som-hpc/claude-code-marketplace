---
name: code-overview
description: Quickly orient in an unfamiliar research repo — map structure, find the analysis entry point and the reproducibility contract (Slurm/uv/renv/manifests). TRIGGER when entering an unfamiliar repo, locating an analysis step, or planning a change.
related:
  - programming-and-coding
  - coding-in-python
  - coding-in-r
  - using-git-and-github
  - starting-a-new-project
updated: 2026-05-22
---
# Code Overview

Goal: map the repo before editing. Do not read everything.

## Learn first

1. What does this project do?
2. Data flow: raw → derived → results.
3. Entry points: scripts, notebooks, Slurm, Makefile/justfile.
4. Load-bearing files.
5. Reproducibility contract: `uv.lock`, `renv.lock`, modules, containers.

## Five-minute walk

Use portable commands. `rg`, `fd`, `tree` are fine if installed; do not require them.

```bash
# pitch
sed -n '1,120p' README.md 2>/dev/null || true
test -f CLAUDE.md && sed -n '1,120p' CLAUDE.md
test -f AGENTS.md && sed -n '1,120p' AGENTS.md

# shape
pwd
ls -la
find . -maxdepth 2 -type d \
  -not -path '*/\.*' \
  -not -path '*/__pycache__*' \
  -not -path '*/.venv*' | sort

# env / ignore contract
sed -n '1,200p' .gitignore 2>/dev/null || true
sed -n '1,120p' pyproject.toml 2>/dev/null || true
sed -n '1,80p' renv.lock 2>/dev/null || true
sed -n '1,120p' DESCRIPTION 2>/dev/null || true

# entry points
test -d scripts && find scripts -maxdepth 2 -type f | sort
test -d slurm && find slurm -maxdepth 2 -type f | sort
test -d notebooks && find notebooks -maxdepth 1 -type f | sort
find . -maxdepth 3 -type f \( -name '*.sh' -o -name '*.sbatch' -o -name '*.slurm' \) | sort
grep -RIl --include='*.py' 'if __name__' . 2>/dev/null | head -20
test -f justfile && sed -n '1,160p' justfile
test -f Makefile && sed -n '1,120p' Makefile

# shared code
test -d src && find src -type f | sort
test -d R && find R -type f | sort
test -f lib.py && wc -l lib.py
```

## Typical shape

```text
project/
├── README.md
├── pyproject.toml + uv.lock
├── renv.lock + .Rprofile
├── scripts/
├── slurm/
├── notebooks/
├── src/ or lib.py or R/
├── data/raw/
├── data/derived/
└── results/
```

## Notebooks

Read lightly first: headings, imports, data-load cells, final save/export cells.

```bash
jupyter nbconvert --to script notebooks/example.ipynb --stdout | sed -n '1,160p'
```

## Stop when you know

- project purpose,
- data flow,
- likely entry points,
- 1–3 files to edit,
- environment and run command.

## Report format

```text
**Project:** one sentence.
**Data flow:** raw → derived → results.
**Entry points:** scripts/build_panel.py; slurm/build_panel.sbatch.
**Load-bearing:** src/panel.py, src/model.py.
**Stack:** uv + Polars; Slurm for full runs.
**Conventions:** uv.lock committed; data/raw ignored.
```

## Further reading

- [`ripgrep`](https://github.com/BurntSushi/ripgrep)
- [`tree`](https://oldmanprogrammer.net/source.php?dir=projects/tree)
- [`tokei`](https://github.com/XAMPPRocky/tokei)
