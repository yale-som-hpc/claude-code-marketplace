---
name: justfile
description: just/justfile task-runner guidance — recipes, variables, make fallback, plus SLURM_CPUS_PER_TASK vars and keeping Slurm resources in sbatch for cluster work. TRIGGER when writing, editing, or debugging a justfile, or choosing just vs make.
related:
  - starting-a-new-project
  - coding-in-python
  - coding-in-r
  - installing-software
updated: 2026-05-22
---
# Justfile

`just` is a nice command runner. It is **not installed by default on SOM HPC**.

## Choose

- Shell script: one task, no extra tool.
- Makefile: maximum availability; `make` exists on HPC.
- Justfile: nicer syntax/help/params; requires installing `just`.

If `just` is critical, document install or provide Make/shell fallback. See [installing software](../installing-software/SKILL.md).

## Boilerplate

```just
set dotenv-load
set shell := ["bash", "-euo", "pipefail", "-c"]

default:
    @just --list
```

## Variables

```just
project := justfile_directory()
data := env_var_or_default("DATA_DIR", project / "data")
threads := env_var_or_default("SLURM_CPUS_PER_TASK", "1")
```

## Recipes

```just
# Format Python
fmt:
    uv run ruff format .

# Run tests
test:
    uv run pytest

# Build panel
build-panel:
    .venv/bin/python scripts/build_panel.py \
      --input data/raw/panel.csv \
      --output data/derived/panel.parquet
```

## Parameters

```just
run TASK_ID:
    .venv/bin/python scripts/run_task.py --task-id {{TASK_ID}}

sample N="1000":
    .venv/bin/python scripts/sample.py --n {{N}}

args *ARGS:
    .venv/bin/python scripts/main.py {{ARGS}}
```

## Slurm

Keep resource requests in `slurm/*.sbatch`; let `just` submit them.

```just
submit:
    sbatch slurm/run.sbatch

queue:
    squeue --me
```

Do not hide important Slurm resources inside recipes.

## Shebang recipe

```just
clean-logs:
    #!/usr/bin/env bash
    set -euo pipefail
    find logs -type f -name '*.out' -mtime +30 -print -delete
```

## Make fallback

```make
.PHONY: help fmt test submit
help:
	@grep -E '^[a-zA-Z_-]+:' Makefile

fmt:
	uv run ruff format .

test:
	uv run pytest

submit:
	sbatch slurm/run.sbatch
```

## Checklist

- [ ] `just --list` useful
- [ ] Slurm resources live in `slurm/*.sbatch`
- [ ] README says how to install `just`, or fallback exists
- [ ] Recipes use project env/lockfiles, not global state

## Further reading

- [just manual](https://just.systems/man/en/)
- [GNU Make manual](https://www.gnu.org/software/make/manual/make.html)
