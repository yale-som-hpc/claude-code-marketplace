---
name: installing-software
description: Install software on the Yale SOM HPC cluster with Lmod modules, uv, static/musl binaries, and Apptainer — no sudo. TRIGGER when installing tools or packages on the Yale SOM HPC cluster, hitting GLIBC errors on cluster nodes, building from source on the cluster, or using Apptainer containers there.
related:
  - overview
  - running-python
  - running-r
  - connecting-securely
  - using-the-filesystem
updated: 2026-05-06
---
# Installing Software

Rule: prefer cluster modules for system software, `uv`/`renv` for language packages, and static binaries in `~/.local/bin` for user tools.

## Modules first

```bash
module avail
module spider git
module spider r
module spider cuda
module load git
module load r
```

Important: Git may require `module load git`. Do not assume it is in the default PATH.

For Python projects, prefer `uv` and a project `.venv` over loading a generic Python module. Use the Python module only when you specifically need cluster-provided Python or a module-provided package.

In reusable shell scripts and Slurm scripts, load required modules explicitly rather than relying on your interactive shell state:

```bash
module purge
module load git
module load r
```

## User binaries

Put user-installed command-line tools here:

```bash
mkdir -p ~/.local/bin ~/go/bin
export PATH="$HOME/.local/bin:$HOME/go/bin:$PATH"
```

Prefer statically-linked builds from GitHub releases if available. Inspect "curl to sh" install patterns prior to using those. Common per-user tools:

- `uv` — Python project manager (install snippet below).
- `duckdb` — CLI for SQL over Parquet/CSV/JSON; not module-loadable on the cluster. See [working with large data](../working-with-large-data/SKILL.md#installing-the-cli-tools).
- `qsv` — fast CSV triage; not module-loadable. Same place.
- `gh` — GitHub CLI.
- `jq` — JSON on the command line.
- `ripgrep` — fast recursive grep.
- `croc`, `rclone` — file transfer (see [using the filesystem](../using-the-filesystem/SKILL.md#moving-files)).


## Prefer static or musl binaries

If a GitHub release offers a Linux musl build, prefer it:

```text
x86_64-unknown-linux-musl
x86_64-musl
static-linux-amd64
```

Why: modern prebuilt binaries often require a newer `glibc` than the cluster provides. Static/musl builds avoid many `GLIBC_2.xx not found` errors.

Typical failure:

```text
./tool: /lib64/libc.so.6: version `GLIBC_2.38' not found
```

Fix: download a musl/static build, use a module, build in Apptainer, or compile on the cluster.

## Install `uv`

Install `uv` once into your user tools directory:

```bash
mkdir -p ~/.local/bin
curl -LsSf https://astral.sh/uv/install.sh | sh
export PATH="$HOME/.local/bin:$PATH"
uv --version
```

Then use it per project:

```bash
cd /gpfs/project/myproject/code
uv init --app
uv add polars pyarrow duckdb
uv sync
```

## GitHub CLI

```bash
module load git
gh auth login
gh repo clone owner/repo
```

If SSH auth fails, see [connecting securely](../connecting-securely/SKILL.md).

## When to use Apptainer

Apptainer is not in the default PATH. Load the module first:

```bash
module load apptainer
```

Use Apptainer when:

- binaries need incompatible system libraries
- software has complex C/CUDA dependencies
- you need reproducibility beyond Python/R lockfiles
- a Docker image already exists


## For Python: use uv

System Python, `pip install --user`, and conda environments all leak state between projects, slow down GPFS, or both. `uv` is the way.

```bash
cd /gpfs/project/myproject/code
uv add polars pyarrow duckdb
uv sync --frozen
```

`uv add` + `uv sync --frozen` at setup time on a login node. Then jobs run `srun .venv/bin/python ...`. Never `pip install --user`, never `conda create -n job_$SLURM_JOB_ID`, never `pip install` inside a Slurm array. Commit `pyproject.toml` and `uv.lock`.

## XDG directories

Keep caches out of crowded home directories when possible:

```bash
export XDG_CACHE_HOME="/gpfs/scratch60/$USER/.cache"
export XDG_CONFIG_HOME="$HOME/.config"
export XDG_DATA_HOME="$HOME/.local/share"
```

Create them:

```bash
mkdir -p "$XDG_CACHE_HOME" "$XDG_CONFIG_HOME" "$XDG_DATA_HOME" ~/.local/bin
```

## Checklist

- [ ] Try `module spider` / `module load` first.
- [ ] User binaries go in `~/.local/bin` or `~/go/bin`.
- [ ] `~/.local/bin` is before system paths in `PATH`.
- [ ] Caches use `XDG_CACHE_HOME` when heavy.
- [ ] Static/musl binaries are preferred for standalone tools.
- [ ] No `sudo`, no credentials in install scripts, no per-job environments.

## Further reading

- [uv documentation](https://docs.astral.sh/uv/) — `uv init`, `uv add`, `uv run`, lockfile and Python interpreter management.
- [Lmod user guide](https://lmod.readthedocs.io/en/latest/) — `module spider`, dependency hierarchy, `module purge`.
- [Apptainer user guide](https://apptainer.org/docs/user/latest/) — building, pulling, running containers without root.
- [XDG Base Directory spec](https://specifications.freedesktop.org/basedir-spec/latest/) — `XDG_CACHE_HOME`/`XDG_CONFIG_HOME`/`XDG_DATA_HOME`.
