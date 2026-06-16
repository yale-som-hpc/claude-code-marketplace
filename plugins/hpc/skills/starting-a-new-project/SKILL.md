---
name: starting-a-new-project
description: Create a research project layout under /gpfs/project on the Yale SOM HPC cluster — reproducible, resumable, safe for shared use. TRIGGER when starting or reorganizing a project on the Yale SOM HPC cluster, choosing GPFS directories, or setting up cluster-side logs, lockfiles, and Slurm scripts.
related:
  - programming-and-coding
  - using-git-and-github
  - task-runner
  - installing-software
  - managing-jobs
  - running-python
  - running-r
  - using-the-filesystem
  - acquiring-data
updated: 2026-06-05
---
# Starting a New Project

Rule: make the project understandable to a new RA and restartable by a future you.

## Recommended layout

```text
/gpfs/project/myproject/
├── code/                 # Git repo
│   ├── README.md
│   ├── CLAUDE.md
│   ├── pyproject.toml
│   ├── uv.lock
│   ├── renv.lock
│   ├── src/
│   ├── scripts/
│   └── slurm/
├── data/
│   ├── raw/              # read-only original data
│   └── derived/          # rebuildable intermediates
├── output/               # tables, figures, final outputs
├── logs/                 # Slurm logs
└── cache/                # API/download/model caches
```

## First commands

For collaborative work, request a shared `/gpfs/project/...` folder from SOM IT instead of coordinating through one person's home directory. Put shared data, scripts, logs, and outputs there so permissions and ownership match the project.

```bash
mkdir -p /gpfs/project/myproject/{code,data/raw,data/derived,output,logs,cache}
cd /gpfs/project/myproject/code
git init
uv init --app
mkdir -p src scripts slurm
```

## `.gitignore`

Track code, lockfiles, and documentation. Do not track data, credentials, environments, logs, or generated output. Pull a starting point from [github/gitignore](https://github.com/github/gitignore) and add `data/`, `logs/`, `output/`, `.env`.

## README minimum

```markdown
# Project Name

## Setup

1. Clone repo into `/gpfs/project/myproject/code`.
2. Run `uv sync` on the login node.
3. If using R, run `Rscript -e 'renv::restore()'` on the login node.
4. Submit `sbatch slurm/test.sh`.

## Pipeline

1. `sbatch slurm/01_prepare_data.sh`
2. `sbatch slurm/02_estimate.sh`
3. Outputs appear in `/gpfs/project/myproject/output/`.

## Data

- `data/raw/`: original data, do not edit.
- `data/derived/`: rebuildable from raw data and code.
```

## Seed a `CLAUDE.md`

Drop a short `CLAUDE.md` in the repo root so the project's cluster norms persist across sessions — Claude Code reads it automatically at the start of every session, and so does a future RA. Keep it to the project's specifics; the installed skills already carry the general rules.

```markdown
# Project notes

- This project runs on the Yale SOM HPC cluster. Follow the `hpc` skills.
- Project root: `/gpfs/project/myproject`. Raw data in `data/raw/` is read-only.
- Run analysis on compute nodes via `sbatch slurm/*.sh`, never on the login node.
- Python env is managed with uv; build it once with `uv sync --frozen`, don't `uv sync` inside jobs.
- After a real job, run `seff` and report whether it was right-sized.
```

Claude Code reads `CLAUDE.md`, **not** `AGENTS.md`. If collaborators also use other agents (Codex, Cursor) that read `AGENTS.md`, keep the shared notes in `AGENTS.md` and make `CLAUDE.md` a one-line pointer to it. Claude Code expands the `@` import at load, so this pulls the whole file in:

```markdown
Read @AGENTS.md for this project's context.
```

(A symlink — `ln -s AGENTS.md CLAUDE.md` — works too if you don't need any Claude-only lines.)

## Capture your commands in a task runner

Put the project's common commands (setup, build, submit) in one file so they're reproducible and the agent can rerun them. **`just` is not installed on the cluster** — start with a `Makefile` or `run.sh`, which work out of the box. See [task runner](../task-runner/SKILL.md) for the full treatment and when to use shell vs make vs just.

```make
.PHONY: setup test-job
setup:
	uv sync

test-job:
	sbatch slurm/test.sh        # resources live in the sbatch file, not here
```

Keep recipes thin — the real logic lives in your Python/R/Stata scripts, not the runner.

## First test job

Always submit a tiny test job before any full run. Use the minimal sbatch template from [managing jobs](../managing-jobs/SKILL.md) with `--time=00:10:00 --mem=2G --cpus-per-task=1`, and have it print `sys.version` and `pathlib.Path.cwd()`. If that fails, fix it before submitting anything larger.

## Make raw data read-only after ingest

After the raw-data ingest is complete and checked, freeze it:

```bash
chmod -R g-w /gpfs/project/myproject/data/raw
```

Do not lock `data/raw/` before collaborators have finished placing the initial files there.

## Checklist

- [ ] Project lives in `/gpfs/project/...`, not one person's home directory.
- [ ] Code is in Git.
- [ ] Raw data is read-only.
- [ ] Environments are reproducible from lockfiles.
- [ ] Logs go to `logs/`.
- [ ] A task runner (`Makefile`/`run.sh`) or README documents the common commands.
- [ ] `CLAUDE.md` records the project's cluster norms (Claude Code reads it, not `AGENTS.md`).
- [ ] First Slurm test job passes before any full run.

## Further reading

- [uv documentation](https://docs.astral.sh/uv/) — `uv init --app`, `uv sync`, `uv add`.
- [renv introduction](https://rstudio.github.io/renv/articles/renv.html) — `renv::init`, `renv::snapshot`, `renv::restore`.
- [Just manual](https://just.systems/man/en/) — recipes, dotenv loading, parameters.
- [github/gitignore](https://github.com/github/gitignore) — canonical `.gitignore` templates per language and tool.
