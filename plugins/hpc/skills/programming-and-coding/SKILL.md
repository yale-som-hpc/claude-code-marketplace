---
name: programming-and-coding
description: Cross-cutting research-coding principles — simplicity (KISS/YAGNI), clarity over cleverness, reproducibility, seeds, never mutating raw data, failing loudly. TRIGGER when writing or reviewing research code and no language- or task-specific skill (coding-in-python, coding-in-r, code-review, running-*) fits.
related:
  - coding-in-python
  - coding-in-r
  - code-review
updated: 2026-06-16
---
# Programming and Coding

Audience: SOM scholars and RAs writing research code. Optimize for correctness, rerunnability, and future-you. This is the fallback when no language- or task-specific skill applies; it carries the reproducibility habits the others assume.

## Reproducibility is correctness

- **Set seeds** for anything stochastic: `np.random.default_rng(42)`, `random_state=42`, `set.seed(42)`.
- **Commit lockfiles** (`uv.lock`, `renv.lock`) and record the inputs a result was built from.
- **Never mutate raw data.** Read from `data/raw/` (read-only), write to `data/derived/` — re-derive, don't edit in place.
- **Fail loudly** on missing inputs, schema mismatches, unexpected row-count changes, or violated assumptions — a silent wrong number is worse than a crash. Catch only specific exceptions you can handle, and log enough context to reproduce the failure.

## Design principles

- **KISS** — prefer the simplest correct solution; if code needs a paragraph to explain, simplify it.
- **YAGNI** — build for the task in front of you, not a hypothetical future. Make the smallest change that works; don't add features, abstractions, or configuration nobody asked for.
- **Clarity over cleverness; explicit over implicit; flat over nested.** Readability beats micro-optimization unless performance is measured and critical.
- **Boring tools first** — stdlib / base language, then established libraries, before anything exotic.

## Write for the next person (often future-you)

- Descriptive names: `firm_panel`, `event_window`, `treatment_group`; not `df2`, `tmp`, `x`.
- Focused functions (~one job), manageable files (split by concern).
- Smoke-test on a small slice of real data before a full run; check the output for plausibility, not just exit status.

## Checklist

- [ ] Seeds set where needed.
- [ ] Lockfiles updated if dependencies changed.
- [ ] Raw data unchanged; outputs written to `derived/`.
- [ ] Code fails loudly on bad inputs / row-count surprises.
- [ ] Ran a small realistic smoke test; output is plausible.
- [ ] Simplest correct approach; no speculative features or abstractions.
- [ ] No hardcoded personal paths.

## Further reading

- [Good enough practices in scientific computing](https://doi.org/10.1371/journal.pcbi.1005510)
- [Ten Simple Rules for Reproducible Computational Research](https://journals.plos.org/ploscompbiol/article?id=10.1371/journal.pcbi.1003285)
