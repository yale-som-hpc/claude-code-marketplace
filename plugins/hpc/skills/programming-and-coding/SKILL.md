---
name: programming-and-coding
description: General research coding rules. TRIGGER when writing, reviewing, refactoring, or debugging code.
related:
  - coding-in-python
  - coding-in-r
  - code-review
  - code-overview
updated: 2026-05-22
---
# Programming and Coding

Audience: SOM scholars and RAs writing research code. Optimize for correctness, rerunnability, and future-you.

## Rules

- Keep it simple. If code needs a paragraph of explanation, simplify it.
- Make small changes. Run a small check after each meaningful change.
- Prefer smoke tests on real data slices over elaborate test scaffolding.
- Use descriptive names: `firm_panel`, `event_window`, `treatment_group`; not `df2`, `tmp`, `x`.
- Keep functions focused. Rough target: one job, ~20 lines.
- Keep files manageable. Rough target: <500 lines; split by concern.
- Use boring tools first: stdlib/base language, then established libraries.
- Set seeds for stochastic work: `random_state=42`, `np.random.default_rng(42)`, `set.seed(42)`.
- Treat reproducibility as correctness: commit lockfiles, record inputs, do not mutate raw data.

## Workflow

1. Read `README.md`, `CLAUDE.md`, `AGENTS.md` if present.
2. Inspect nearby code. Match local style unless it is unsafe.
3. Make the smallest correct change.
4. Run a realistic small-input check.
5. Check the output for plausibility, not just exit status.
6. Remove dead code and stale comments.

## Error handling

- Catch only errors you can handle.
- Catch specific exception classes.
- Fail loudly on missing inputs, schema mismatches, unexpected row drops, invalid assumptions.
- Log enough context to reproduce the failure.

## Review mindset

- Clarity over cleverness.
- Explicit over implicit.
- Flat over nested.
- Readability over micro-optimization unless performance is proven critical.

## Checklist

- [ ] Small realistic smoke test run
- [ ] No hardcoded personal paths
- [ ] Seeds set where needed
- [ ] Lockfiles updated if dependencies changed
- [ ] Raw data unchanged
- [ ] Output plausible

## Further reading

- [Good enough practices in scientific computing](https://doi.org/10.1371/journal.pcbi.1005510)
- [Ten Simple Rules for Reproducible Computational Research](https://journals.plos.org/ploscompbiol/article?id=10.1371/journal.pcbi.1003285)
