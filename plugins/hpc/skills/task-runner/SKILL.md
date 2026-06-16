---
name: task-runner
description: Capture a project's commands as named, rerunnable recipes — via shell script, Makefile, or justfile — so analysis steps are reproducible and the agent can rerun them. TRIGGER when automating or documenting project commands, writing a justfile/Makefile/run script, or choosing among shell vs make vs just.
related:
  - starting-a-new-project
  - coding-in-python
  - coding-in-r
  - managing-jobs
  - installing-software
updated: 2026-06-10
---
# Task Runner

Rule: capture a project's commands as named, rerunnable recipes in one file, so a future you — and the agent — can rerun any analysis step without re-deriving the incantation. This is about reproducibility, not convenience.

A research pipeline is a sequence of steps (prepare → estimate → tables). If those commands live only in shell history or someone's head, the work isn't reproducible and the agent can't reliably rerun it. Put them in one discoverable file with named recipes; the agent will read that file and reuse the commands instead of guessing.

## Pick a tool — what's installed on the cluster matters

- **Shell script (`run.sh`)** — zero dependencies, always present. Best for a short linear pipeline; a `case "$1" in ... esac` dispatch gives you named steps.
- **Makefile** — `make` is installed on the cluster **by default**. Good when steps have dependencies or you want skip-if-already-built behavior. Mind the tab indentation and declare `.PHONY` targets.
- **justfile** — the nicest ergonomics (`just --list`, parameters, dotenv), **but `just` is NOT installed on the cluster and is not a module** — you must install it yourself (`cargo install just`, or drop a static binary in `~/.local/bin`; see [installing software](../installing-software/SKILL.md)). Use it only if you'll install it.

Default for someone new: a **Makefile or `run.sh`** — they work the moment you log in. Reach for `just` once you've installed it and want the ergonomics.

## Cluster integration

- **Keep Slurm resources in `slurm/*.sbatch`, not in the runner.** The runner only *submits* (`sbatch slurm/run.sbatch`); never hide `--mem`/`--cpus-per-task`/`--time` where a reader won't see them.
- **Read the allocation, don't hardcode it:** `${SLURM_CPUS_PER_TASK:-1}` in shell, `env_var_or_default("SLURM_CPUS_PER_TASK", "1")` in just, `$(or $(SLURM_CPUS_PER_TASK),1)` in make.
- Recipes should call the project env / lockfiles (`.venv/bin/python`, `Rscript`), not global state.

## Shell script (always available)

```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")"

case "${1:-help}" in
  setup)   uv sync ;;
  build)   .venv/bin/python scripts/build_panel.py ;;
  submit)  sbatch slurm/run.sbatch ;;
  *)       echo "usage: ./run.sh {setup|build|submit}" ;;
esac
```

## Makefile (default-available on the cluster)

```make
.PHONY: help setup build submit
THREADS ?= $(or $(SLURM_CPUS_PER_TASK),1)

help:
	@grep -E '^[a-zA-Z_-]+:' Makefile

setup:
	uv sync

build:
	.venv/bin/python scripts/build_panel.py --threads $(THREADS)

submit:
	sbatch slurm/run.sbatch        # resources live in the sbatch file, not here
```

## justfile (install `just` first)

```just
set dotenv-load
set shell := ["bash", "-euo", "pipefail", "-c"]
threads := env_var_or_default("SLURM_CPUS_PER_TASK", "1")

default:
    @just --list

setup:
    uv sync

build *ARGS:
    .venv/bin/python scripts/build_panel.py --threads {{threads}} {{ARGS}}

submit:
    sbatch slurm/run.sbatch
```

Keep recipes thin — the real logic belongs in your Python/R/Stata scripts, not the runner.

## Checklist

- [ ] Common commands captured in one file (`run.sh`, `Makefile`, or `justfile`).
- [ ] Tool choice matches what's installed: `make`/shell work out of the box; `just` needs a user install.
- [ ] Slurm resources live in `slurm/*.sbatch`, not hidden in recipes.
- [ ] Thread counts read `SLURM_CPUS_PER_TASK`, not hardcoded.
- [ ] Recipes use the project env/lockfiles, not global state.

## Further reading

- [GNU Make manual](https://www.gnu.org/software/make/manual/make.html) — targets, `.PHONY`, variables, pattern rules.
- [just manual](https://just.systems/man/en/) — recipes, dotenv loading, parameters.
