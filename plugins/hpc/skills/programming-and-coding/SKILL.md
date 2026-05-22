---
name: programming-and-coding
description: General coding philosophy for research code — KISS, small functions, smoke tests on real data, seeds, and readable analysis scripts. TRIGGER when writing, reviewing, refactoring, or debugging code in any language for Yale SOM research work, whether on a laptop or the HPC cluster.
related:
  - coding-in-python
  - coding-in-r
  - code-review
  - code-overview
updated: 2026-05-22
---
# Programming and Coding

Cross-language baseline. Load this alongside `coding-in-python`, `coding-in-r`, `running-python`, or `running-r` as the *why*; those skills cover the *what*.

The audience is SOM scholars and RAs writing research code in finance, economics, OB, OR, accounting, marketing, and related fields. The bar is: correct enough to publish from, clear enough for a future RA to rerun, and simple enough to debug under deadline pressure.

## Core principles

- **KISS** — abhor complexity. If a function is hard to understand, simplify before adding comments.
- **Incremental changes** — small, testable edits. Rerun a small slice after every meaningful change.
- **Smoke tests on real data beat theoretical tests.** For analysis code, a small realistic input catches the bugs that matter.
- **Working code is the documentation.** Use good names and focused functions. Comments explain why, not what.
- **Reproducibility is part of correctness.** Commit lockfiles, set seeds, and keep raw data immutable.

## Code structure

- Small, focused functions. Rough target: ~20 lines, one responsibility.
- Descriptive names: `firm_panel`, `event_window`, `treatment_group`, not `df2` or `tmp`.
- Use the language's type system where it costs little: Python type hints on function signatures; R assertions for important assumptions.
- Boring, proven solutions first: stdlib/base language, then established libraries. Avoid novel frameworks for one-off analyses.
- Keep files small. Aim for <500 lines. A giant script usually wants `scripts/` entry points plus shared helpers in `src/`, `lib.py`, or `R/`.
- Set seeds on stochastic operations: `np.random.default_rng(42)`, `random_state=42`, `set.seed(42)`.

## How to work on a task

1. Read the request plus `README.md`, `CLAUDE.md`, `AGENTS.md`, and project docs if present.
2. Map the existing code before editing. Match local conventions unless they violate reproducibility or cluster safety.
3. Make the smallest change that could work.
4. Run a smoke test on a realistic small input.
5. Check outputs for plausibility, not just successful exit status.
6. Clean up dead code and stale comments before stopping.

## Testing in moderation

- For research code, one trustworthy end-to-end smoke test often beats broad unit-test coverage.
- Unit-test formulas, data transforms, simulation helpers, and boundary conditions that are hard to eyeball.
- Skip tests for throwaway scripts, but still run the script on a tiny input before trusting it.

## Error handling

- Handle errors only where you can recover meaningfully.
- Catch specific exception classes. Never bare `except:` or `tryCatch(error = function(e) NULL)`.
- Fail loudly on missing inputs, schema mismatches, unexpected row drops, or invalid assumptions.
- Include enough context in errors/logs to reproduce the failure.

## Review mindset

- Clarity over cleverness.
- Explicit over implicit.
- Flat over nested.
- Readability over micro-optimization unless performance is proven critical.
- See [code review](../code-review/SKILL.md) before committing or publishing analysis code.

## Checklist

- [ ] Small realistic smoke test run
- [ ] No hardcoded personal paths
- [ ] Seeds set where needed
- [ ] Lockfiles updated if dependencies changed
- [ ] Raw data unchanged
- [ ] Output looks plausible, not merely produced

## Further reading

- [The Pragmatic Programmer](https://pragprog.com/titles/tpp20/the-pragmatic-programmer-20th-anniversary-edition/)
- [Ten Simple Rules for Reproducible Computational Research](https://journals.plos.org/ploscompbiol/article?id=10.1371/journal.pcbi.1003285)
- [Good enough practices in scientific computing](https://doi.org/10.1371/journal.pcbi.1005510)
