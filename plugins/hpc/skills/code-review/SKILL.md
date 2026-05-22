---
name: code-review
description: Code review for research repos — find real bugs, reproducibility failures, hardcoded paths, staged data, missing seeds, lockfile drift, and HPC resource mistakes. TRIGGER when reviewing code, diffs, PRs, or implementations; before committing research code; or when asked whether a change is safe to run or publish.
related:
  - programming-and-coding
  - coding-in-python
  - coding-in-r
  - using-git-and-github
  - running-python
  - running-r
updated: 2026-05-22
---
# Code Review

Find bugs that would break the analysis, waste cluster resources, leak data, or make results unreproducible. Not style nits.

## A finding must be

1. **Introduced in this change** — not pre-existing unless the user asked for a broad audit.
2. **Provably impactful** — name the scenario or input that triggers it.
3. **Actionable** — there is a concrete fix.

## Severity

- **P0** — Blocks commit/merge/run. Wrong answer, lost data, secret leaked, code cannot run elsewhere, serious cluster misuse.
- **P1** — Should fix before commit/merge. Realistic inputs trigger it.
- **P2** — Worth fixing eventually. Low probability or limited blast radius.
- **P3** — Nit. Do not report unless explicitly asked.

End every review with exactly one verdict:

- **LGTM** — no P0/P1. Ship it.
- **FAIL** — P0/P1 found. Must fix first.

## Research-code landmines

Always check:

- **Hardcoded personal paths.** `/Users/...`, `C:\Users\...`, `/home/netid/...`, `/gpfs/scratch60/netid/...`. **P0** if it prevents another researcher or compute node from running the code.
- **Raw data, large outputs, secrets, or environments staged.** `data/raw/`, `.env`, keys, `.venv/`, `renv/library/`, large Parquet/CSV/RDS outputs. **P0** for secrets or large/raw data.
- **Missing seeds.** Simulation, bootstrap, train/test split, random forest, UMAP, stochastic optimizer, parallel RNG without a fixed plan. **P1**.
- **Silent data loss.** `dropna()`, `na.omit()`, broad filters, failed joins, or duplicate drops without counts before/after. **P1**.
- **Lockfile drift.** Python dependency added without `uv.lock`; R package used without `renv.lock`; Slurm job relies on untracked environment state. **P0/P1**.
- **Mutable raw data.** Code writes into `data/raw/` or overwrites original inputs. **P0**.
- **HPC resource mismatch.** Python/R uses all cores while Slurm requested one, or job requests a GPU but code never uses it. **P1**.
- **GPFS tiny-file storm.** Arrays writing thousands of tiny files instead of chunked outputs, Parquet, SQLite, or one output per task. **P1**.
- **Package installs inside arrays.** Many workers running `uv sync`, `install.packages`, or `renv::restore()` concurrently. **P1**.
- **Fragile scraping/API behavior.** No rate limit, no cache, no retry/backoff, or many workers stampeding the same endpoint. **P1**.

## Process

1. Read changed files end-to-end with the `read` tool.
2. Check `git status --short` and staged files.
3. Run the project’s existing checks if practical.
4. Run a small realistic smoke test if data and runtime allow.
5. For Slurm changes, inspect requested resources, thread controls, paths, logs, and resumability.

## Output format

```text
## Findings

### [P1] Missing seed makes bootstrap estimates non-reproducible
**File**: scripts/bootstrap.py:42
**Issue**: `np.random.default_rng()` is called without a fixed seed, so confidence intervals change across runs.
**Fix**: create `rng = np.random.default_rng(42)` at the entry point and pass it into the bootstrap helper.

## Verdict
FAIL
```

If no P0/P1:

```text
## Findings
None.

## Verdict
LGTM
```

## Checklist

- [ ] Changed files read fully
- [ ] Staged/untracked files checked for data, secrets, and environment dirs
- [ ] Seeds checked
- [ ] Lockfiles checked
- [ ] Small smoke test or existing checks run when practical
- [ ] Slurm/resource behavior checked for cluster code

## Further reading

- [Google code review guide](https://google.github.io/eng-practices/review/reviewer/)
- [Good enough practices in scientific computing](https://doi.org/10.1371/journal.pcbi.1005510)
