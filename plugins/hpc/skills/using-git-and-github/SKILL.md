---
name: using-git-and-github
description: Git and GitHub guidance for AI agents helping researchers — commit useful checkpoints, avoid raw data and secrets, handle branches/PRs pragmatically, and keep repos reproducible. TRIGGER when running git or gh, creating commits, deciding what to track or ignore, opening PRs, creating repos, or helping a Yale SOM researcher use version control.
related:
  - programming-and-coding
  - code-review
  - coding-in-python
  - coding-in-r
  - starting-a-new-project
updated: 2026-05-22
---
# Using Git and GitHub

This skill is for agents helping researchers who want version-control benefits without becoming git experts.

## First principles

- Git tracks code, configuration, metadata, small examples, and reproducibility contracts.
- Git does **not** track raw data, secrets, virtual environments, package libraries, or large generated outputs.
- Commit logical checkpoints. Researchers should be able to recover yesterday's working analysis.
- Prefer simple history. No force-push on shared branches unless the user explicitly understands the risk.
- Explain git decisions in plain language.

## Agent behavior

- Check `git status --short` before and after edits.
- Do not silently stage ambiguous untracked files. Report them and propose track/ignore/refuse.
- Commit when the user asks, or when a coherent requested task is complete and the repo appears to expect agent commits. If uncertain, ask once.
- Use clear imperative commit messages: `Add panel construction script`, `Fix Slurm memory request`, `Document WRDS cache layout`.
- Surface the commit hash and summary.

## What to track

Usually track:

- source code, scripts, Slurm scripts,
- README and project docs,
- `pyproject.toml`, `uv.lock`, `renv.lock`, `.Rprofile`,
- small sample metadata / codebooks / config files,
- tiny synthetic fixtures used for tests,
- Makefiles and justfiles.

Usually ignore:

```gitignore
# Data and generated outputs
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

Adjust per project. For example, final manuscript figures may belong in `paper/figures/`, while large rebuildable `results/` do not.

## Big-file pushback

Before committing, inspect staged sizes:

```bash
git diff --cached --name-only | while read -r f; do
  test -f "$f" && wc -c "$f"
done | sort -n
```

Rules of thumb:

- **>5 MB**: warn and ask whether it is generated/data.
- **>50 MB**: refuse unless the user gives a strong reason and the repo policy allows it.
- Never suggest Git LFS as a reflex. For research data, a documented filesystem path, archive DOI, or object store is often better.

Track a `data.yml` or `DATA.md` describing where large data live:

```yaml
- name: crsp_panel.parquet
  location: /gpfs/project/myproject/data/derived/crsp_panel.parquet
  size_bytes: 1234567890
  description: Rebuildable CRSP monthly panel created by scripts/build_panel.py
```

## Branches and PRs

Default for solo/small research repos: commit on `main`.

Use a branch when:

- multiple people actively commit to the repo,
- the change is experimental,
- the user asks for a PR,
- the change is large enough that review matters.

Never force-push a branch others may use. Prefer a revert commit.

## GitHub CLI

If `gh` is available:

```bash
gh auth status
gh repo view --web
gh pr create --fill
```

If not installed on the HPC, do not block the research task. Git over HTTPS/SSH and the web UI are fine. See [installing software](../installing-software/SKILL.md) for user-installed tools.

## Checklist before commit

- [ ] `git status --short` reviewed
- [ ] No raw data, secrets, env dirs, or large generated files staged
- [ ] Lockfiles updated when dependencies changed
- [ ] Relevant smoke test/check run when practical
- [ ] Commit message plain and specific

## Further reading

- [Pro Git](https://git-scm.com/book/en/v2)
- [GitHub CLI manual](https://cli.github.com/manual/)
- [GitHub docs: ignoring files](https://docs.github.com/en/get-started/git-basics/ignoring-files)
