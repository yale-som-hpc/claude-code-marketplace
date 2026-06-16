---
name: code-review
description: Review code, diffs, or PRs for correctness, plus research/cluster pitfalls — hardcoded scratch paths, resource mismatch, GPFS tiny-file storms, package installs inside arrays. TRIGGER when reviewing code or a diff/PR, or before committing.
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

Find bugs that affect results, reproducibility, data safety, or cluster use. Skip style nits.

## Findings must be

- New in this change, unless asked for a broad audit.
- Impactful: name the input/scenario.
- Actionable: give the fix.

## Severity

- **P0**: blocks commit/run/merge. Wrong result, lost data, secret leak, cannot rerun, serious cluster misuse.
- **P1**: fix before commit/merge. Realistic failure.
- **P2**: fix later.
- **P3**: nit. Do not report unless asked.

Verdict:

- **LGTM**: no P0/P1.
- **FAIL**: P0/P1 found.

## Always check

- Hardcoded personal paths: `/Users/...`, `C:\Users\...`, `/home/netid/...`, `/gpfs/scratch60/netid/...`.
- Raw data, secrets, env dirs, or large outputs staged.
- Missing seeds in simulation, bootstrap, train/test split, UMAP, random forest, stochastic optimizer.
- Silent data loss: `dropna()`, `na.omit()`, failed joins, duplicate drops, broad filters without counts.
- Lockfile drift: deps changed but `uv.lock` / `renv.lock` not updated.
- Raw data mutation: code writes into `data/raw/` or overwrites inputs.
- HPC resource mismatch: requested 1 CPU but uses all cores; requested GPU but code does not use it.
- GPFS tiny-file storms.
- Package installs inside arrays.
- Scraping/API code without rate limit, cache, retry/backoff.

## Process

1. Read changed files fully.
2. Check `git status --short` and staged files.
3. Run existing checks if practical.
4. Run a small realistic smoke test if practical.
5. For Slurm changes: check resources, thread env vars, paths, logs, resumability.

## Output

```text
## Findings

### [P1] Missing seed makes bootstrap estimates non-reproducible
**File**: scripts/bootstrap.py:42
**Issue**: `np.random.default_rng()` is called without a fixed seed.
**Fix**: create `rng = np.random.default_rng(42)` at the entry point and pass it in.

## Verdict
FAIL
```

If clean:

```text
## Findings
None.

## Verdict
LGTM
```

## Checklist

- [ ] Changed files read
- [ ] Staged/untracked files checked
- [ ] Seeds checked
- [ ] Lockfiles checked
- [ ] Smoke test/check run if practical
- [ ] Slurm/resource behavior checked if relevant

## Further reading

- [Google code review guide](https://google.github.io/eng-practices/review/reviewer/)
