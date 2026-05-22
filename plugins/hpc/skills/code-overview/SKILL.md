---
name: code-overview
description: Quickly orient inside an unfamiliar research repo before editing — find entry points, data flow, environments, Slurm scripts, and load-bearing files without reading everything. TRIGGER when entering a repo for the first time, asked what a codebase does, planning a change in unfamiliar research code, or locating where an analysis step happens.
related:
  - programming-and-coding
  - coding-in-python
  - coding-in-r
  - using-git-and-github
  - starting-a-new-project
updated: 2026-05-22
---
# Code Overview

Map the repo in five focused minutes. Do not burn context reading every file before you know which files matter.

## What to learn first

1. What does this project do?
2. What is the data flow: raw → derived → results?
3. What are the entry points: scripts, notebooks, Slurm jobs, Justfile/Makefile recipes?
4. Where is the load-bearing logic?
5. How is the environment reproduced: `uv.lock`, `renv.lock`, modules, containers?

## Five-minute walk

Use portable commands that exist almost everywhere. If `rg`, `fd`, or `tree` are installed, they are fine; do not require them.

```bash
# 1. The pitch
sed -n '1,120p' README.md 2>/dev/null || true
test -f CLAUDE.md && sed -n '1,120p' CLAUDE.md
test -f AGENTS.md && sed -n '1,120p' AGENTS.md

# 2. Shape
pwd
ls -la
find . -maxdepth 2 -type d \
  -not -path '*/\.*' \
  -not -path '*/__pycache__*' \
  -not -path '*/.venv*' | sort

# 3. Tracked vs ignored and environment contracts
sed -n '1,200p' .gitignore 2>/dev/null || true
sed -n '1,120p' pyproject.toml 2>/dev/null || true
sed -n '1,80p' renv.lock 2>/dev/null || true
sed -n '1,120p' DESCRIPTION 2>/dev/null || true

# 4. Entry points
test -d scripts && find scripts -maxdepth 2 -type f | sort
test -d slurm && find slurm -maxdepth 2 -type f | sort
test -d notebooks && find notebooks -maxdepth 1 -type f | sort
find . -maxdepth 3 -type f \( -name '*.sh' -o -name '*.sbatch' -o -name '*.slurm' \) | sort
grep -RIl --include='*.py' 'if __name__' . 2>/dev/null | head -20
test -f justfile && sed -n '1,160p' justfile
test -f Makefile && sed -n '1,120p' Makefile

# 5. Shared code
test -d src && find src -type f | sort
test -d R && find R -type f | sort
test -f lib.py && wc -l lib.py
```

## Typical research repo shape

```text
project/
├── README.md
├── CLAUDE.md / AGENTS.md
├── pyproject.toml + uv.lock      # Python
├── renv.lock + .Rprofile         # R
├── scripts/                      # batch entry points
├── slurm/                        # sbatch wrappers
├── notebooks/                    # exploration and figures
├── src/ or lib.py or R/          # shared helpers
├── data/raw/                     # immutable, usually ignored
├── data/derived/                 # rebuildable
└── results/ or output/           # figures, tables, logs
```

## Notebooks

Read notebooks lightly at first:

- title / markdown headings,
- import/setup cells,
- data-load cells,
- final save/export cells.

Convert to text for scanning when useful:

```bash
jupyter nbconvert --to script notebooks/example.ipynb --stdout | sed -n '1,160p'
```

## When to stop reading

Stop once you can say:

- project purpose in one sentence,
- data-flow shape,
- likely entry point(s),
- 1–3 files that need editing,
- environment and run command.

Then edit. Read deeper only for the files touched by the change.

## Report format

```text
**Project:** one sentence.
**Data flow:** raw → derived → results.
**Entry points:** scripts/build_panel.py; slurm/build_panel.sbatch.
**Load-bearing files:** src/panel.py, src/model.py.
**Stack:** uv + Polars; Slurm for full runs.
**Conventions:** uv.lock committed; data/raw ignored; results rebuilt by just/make.
```

## Checklist

- [ ] README / agent instructions read
- [ ] Environment files identified
- [ ] Entry points identified
- [ ] Data and output dirs identified
- [ ] Load-bearing files identified before broad reading

## Further reading

- [`tree`](https://oldmanprogrammer.net/source.php?dir=projects/tree) — useful if installed
- [`ripgrep`](https://github.com/BurntSushi/ripgrep) — fast search if installed
- [`tokei`](https://github.com/XAMPPRocky/tokei) — codebase size by language
