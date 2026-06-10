---
name: using-git-and-github
description: Git and GitHub for work on the Yale SOM HPC cluster — module load git, SSH agent forwarding, research-data .gitignore, big-file handling. TRIGGER when running git/gh on the cluster, committing, deciding what to track, or opening PRs from cluster work.
related:
  - programming-and-coding
  - code-review
  - coding-in-python
  - coding-in-r
  - starting-a-new-project
  - installing-software
  - connecting-securely
updated: 2026-05-22
---
# Using Git and GitHub

For agents helping researchers get version-control benefits without becoming git experts.

## Rules

- Track code, config, metadata, docs, lockfiles, small examples.
- Do not track raw data, secrets, env dirs, package libraries, large generated outputs.
- Commit logical checkpoints.
- Prefer simple history. No force-push on shared branches unless explicitly approved.
- Explain git decisions plainly.

## Git on HPC

Git may require a module:

```bash
module load git
git --version
```

Use `module load git` in setup notes/scripts that need git on HPC.

## Auth and pushing

Laptop: HTTPS with `gh auth login` or normal SSH keys are fine.

HPC: prefer SSH agent forwarding so private keys stay on the laptop:

```bash
ssh -A hpc
module load git
ssh -T git@github.com
git push
```

If `SSH_AUTH_SOCK` is stale, fix the socket pointer. Do **not** copy private keys to the cluster. See [connecting securely](../connecting-securely/SKILL.md).

HTTPS / `gh` token auth is also fine if already configured:

```bash
module load git
gh auth status
git push
```

Never copy `~/.ssh/id_*` private keys onto the cluster. Public keys (`*.pub`) are safe; private keys are not.

## Agent behavior

- Run `git status --short` before and after edits.
- Do not silently stage ambiguous untracked files. Report and propose track/ignore/refuse.
- Commit when asked, or when a coherent requested task is complete and repo norms expect commits. If uncertain, ask once.
- Use imperative messages: `Add panel construction script`, `Fix Slurm memory request`.
- Report hash + summary.

## Track / ignore

Track:

- source, scripts, Slurm scripts,
- README/docs,
- `pyproject.toml`, `uv.lock`, `renv.lock`, `.Rprofile`,
- small metadata/codebooks/config,
- tiny synthetic fixtures,
- Makefiles/justfiles.

Ignore:

```gitignore
# Data and outputs
data/raw/
data/derived/
output/
outputs/
results/
logs/
*.parquet
*.rds
*.RDS
*.dta
*.sas7bdat
*.feather
*.h5
*.h5ad

# Python
.venv/
__pycache__/
*.pyc
.pytest_cache/
.ruff_cache/

# R
.Rhistory
.RData
.Rproj.user/
renv/library/
renv/local/
renv/staging/

# Secrets / local config
.env
.env.local
*.pem
*.key

# OS/editor
.DS_Store
Thumbs.db
```

Adjust per project. Final manuscript figures may belong in `paper/figures/`; rebuildable `results/` usually does not.

## Big files

Before committing:

```bash
git diff --cached --name-only | while read -r f; do
  test -f "$f" && wc -c "$f"
done | sort -n
```

Rules:

- >5 MB: warn and ask.
- >50 MB: refuse unless strong reason and repo policy allow it.
- Do not suggest Git LFS reflexively. Prefer documented filesystem path, DOI, or object store.

Track a pointer file instead:

```yaml
- name: crsp_panel.parquet
  location: /gpfs/project/myproject/data/derived/crsp_panel.parquet
  size_bytes: 1234567890
  description: Rebuildable panel from scripts/build_panel.py
```

## Branches

Default for solo/small research repos: commit on `main`.

Use a branch when multiple people are active, the change is experimental, the user asks, or review matters.

Never force-push a shared branch. Prefer revert commits.

## GitHub CLI

```bash
gh auth status
gh repo view --web
gh pr create --fill
```

If `gh` is missing on HPC, do not block. Use git + web UI.

## Checklist

- [ ] `module load git` used on HPC if needed
- [ ] `git status --short` reviewed
- [ ] No raw data, secrets, env dirs, or large outputs staged
- [ ] Lockfiles updated if deps changed
- [ ] Smoke test/check run if practical
- [ ] Plain specific commit message
- [ ] No private SSH keys copied to HPC

## Further reading

- [Pro Git](https://git-scm.com/book/en/v2)
- [GitHub CLI manual](https://cli.github.com/manual/)
