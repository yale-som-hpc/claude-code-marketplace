---
name: justfile
description: Justfile syntax and task-runner guidance for research projects — when to use just vs make or shell scripts, common recipes, variables, parameters, and safe defaults. TRIGGER when writing, editing, or debugging a justfile; migrating repeated commands into task recipes; or discussing whether to use just, make, or shell scripts in a Yale SOM research project.
related:
  - starting-a-new-project
  - coding-in-python
  - coding-in-r
  - installing-software
updated: 2026-05-22
---
# Justfile

`just` is a pleasant command runner. It is **not installed by default on the SOM HPC cluster**. Use it when the project can install user tools or when collaborators already have it. Use `make` or plain shell scripts when zero extra tooling matters.

## When to use what

- **Shell script**: one task, mostly bash, should run everywhere.
- **Makefile**: maximum availability; `make` exists on most systems including HPC.
- **Justfile**: nicer syntax, good help text, good parameter handling; requires installing `just`.

If you choose `just` on HPC, install it as a user binary in `~/.local/bin` or project `bin/`; see [installing software](../installing-software/SKILL.md). Do not require `just` for a critical cluster workflow unless setup docs say how to install it.

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
# Show available tasks
default:
    @just --list

# Format Python code
fmt:
    uv run ruff format .

# Run tests
test:
    uv run pytest

# Build one output
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

## Slurm wrappers

Keep Slurm resource requests in `slurm/*.sbatch`; let `just` submit them.

```just
submit:
    sbatch slurm/run.sbatch

queue:
    squeue --me
```

Do not hide important resource requests inside a justfile command where collaborators cannot find them.

## Shebang recipes

```just
clean-logs:
    #!/usr/bin/env bash
    set -euo pipefail
    find logs -type f -name '*.out' -mtime +30 -print -delete
```

## Makefile fallback

If `just` is not available, this is often enough:

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

- [ ] `just --list` shows useful recipe descriptions
- [ ] Critical Slurm resources live in `slurm/*.sbatch`, not hidden recipes
- [ ] README says how to install `just`, or Makefile/shell fallback exists
- [ ] Recipes use project lockfiles/environments, not global state

## Further reading

- [just manual](https://just.systems/man/en/)
- [just README](https://github.com/casey/just)
- [GNU Make manual](https://www.gnu.org/software/make/manual/make.html)
