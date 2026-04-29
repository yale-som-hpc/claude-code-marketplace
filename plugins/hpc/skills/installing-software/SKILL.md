---
name: installing-software
description: Use modules, local binaries, uv, XDG directories, musl builds, and Apptainer without sudo. TRIGGER when installing packages/tools, needing unavailable software, building from source, or using containers on the cluster.
related:
  - overview
  - running-python
  - running-r
  - connecting-securely
  - using-the-filesystem
updated: 2026-04-28
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

Good candidates:

- `uv`
- `gh`
- `jq`
- `ripgrep`
- `croc`
- `rclone`

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

Use Apptainer when:

- binaries need incompatible system libraries
- software has complex C/CUDA dependencies
- you need reproducibility beyond Python/R lockfiles
- a Docker image already exists


## Avoid conda by default

Conda can work, but it is usually a poor default on shared GPFS: environments contain many small files, solves can be slow, and users often create one-off environments that are hard to reproduce. Prefer:

- `uv` for Python projects
- modules for cluster-provided system software
- Apptainer for complex binary stacks
- static/musl binaries for standalone tools

Avoid these patterns:

```bash
sudo apt install something        # no sudo
conda create -n job_$SLURM_JOB_ID # inode bomb, not reproducible
pip install --user package        # hard to reproduce
```

Prefer project lockfiles and local binary installs.

## Checklist

- [ ] Try `module spider` / `module load` first.
- [ ] User binaries go in `~/.local/bin` or `~/go/bin`.
- [ ] `~/.local/bin` is before system paths in `PATH`.
- [ ] Caches use `XDG_CACHE_HOME` when heavy.
- [ ] Static/musl binaries are preferred for standalone tools.
- [ ] No `sudo`, no credentials in install scripts, no per-job environments.
