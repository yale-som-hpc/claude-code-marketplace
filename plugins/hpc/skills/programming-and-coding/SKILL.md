---
name: programming-and-coding
description: Cross-cutting research-coding principles. TRIGGER when writing or reviewing research code.
related:
  - coding-in-python
  - coding-in-r
  - code-review
updated: 2026-06-16
---
# Programming and Coding

## Core principles

- **KISS** — prefer the simplest correct solution; if code needs a paragraph to explain, simplify it.
- **YAGNI** — build for the task in front of you, not a hypothetical future. Make the smallest change that works; don't add features, abstractions, or configuration nobody asked for.
- **Clarity over cleverness; explicit over implicit; flat over nested.** Readability beats micro-optimization unless performance is measured and critical.
- **DRY carefully** — DRY is about duplicated knowledge, not similar-looking code. Prefer simple duplication over premature coupling.
- **Boring tools first** — stdlib / base language, then established libraries, before anything exotic.
- **Incremental changes** — baby steps. Small, testable changes that do the minimal work.
- **Working code is documentation** — express intent through names, types, and structure; comments explain why.
- **Premature optimization is the root of all evil** - make it run, make it correct, make it fast, in that order.
- **Be super careful where code could generate costs**, particularly hitting remote resources and subscriptions.

## Code structure and style

- **Use descriptive names**: `firm_panel` instead of  `df2`, `event_window` instead of `tmp`, `treatment_group` instead of `x`, `adminUserEmail` instead of `email`.
- **Write small, focused functions** (~20 lines). One thing per function. Prefer pure functions: same input, same output.
- **Use types, classes, and structures** to make impossible states impossible (or difficult) to represent.
- **Keep files small**. Check with `wc -l`. Aim for <500 lines.
- **Prefer deep, useful modules** with simple interfaces over many shallow abstractions.
- **Name intermediate conditions**; optimize for debugging and reader working memory, not fewest lines.
- **Make state explicit, small, and owned**. Avoid ambient globals, hidden mutation, and action-at-a-distance.
- **Prefer immutable data if convenient**.  Otherwise localize mutations to where ownership is obvious and prefer copy-at-boundaries if practical.
- **Prefer side effects at the edges**: CLI, HTTP, DB, filesystem, clock, randomness, network.

## Reproducibility is correctness

- **Set seeds** for anything stochastic: `np.random.default_rng(42)`, `random_state=42`, `set.seed(42)`.
- **Commit lockfiles** (`uv.lock`, `renv.lock`) and record the inputs a result was built from.
- **Never mutate raw data.** Read from `data/raw/` (read-only), write to `data/derived/` — re-derive, don't edit in place.
- **Manifest derived data.** Write a manifest beside any derived dataset so its provenance travels with the data — so anyone, including the agent in a later session, can answer *what is this, how was it made, is it still valid?* without rerunning the pipeline or guessing. Capture enough to make staleness checkable, using judgment about the specifics (a git commit only pins the code if the working tree was clean). A rough manifest beats none.
- **Fail loudly** on missing inputs, schema mismatches, unexpected row-count changes, or violated assumptions — a silent wrong number is worse than a crash. Catch only specific exceptions you can handle, and log enough context to reproduce the failure.
- **Don't hard-code personal paths** - put defaults in `.env`s, runners (Justfiles, Makefiles, or shell scripts), or config files (prefer TOML, YAML, and JSON).

## Code Review Mindset

- Clarity over cleverness
- Explicit over implicit
- Simple over complex
- Flat over nested
- Boring over novel
- Local behavior over scattered indirection
- Readability over performance (unless performance is measured and proven critical)
- Facts over taste; code health over perfection

## Testing

- Write smoke tests with real, reasonably small data samples. Don't just write unit tests.
- Test in moderation — critical paths, regressions, integration points, not vanity coverage metrics.
- Test boundaries, happy path, likely failure modes.
- Prefer tests that survive refactors; avoid tests that merely freeze implementation.
- Use dependency injection and mocking where tests might hit APIs or other gnarly dependencies.

## Research and Runtime Hygiene

- Separate code, inputs, parameters/configs, and outputs. Do not switch datasets, thresholds, model specs, or output names by editing constants in code.
- Provide one obvious way to run: CLI, config, `make` or `just` recipe. Notebooks must run top-to-bottom; reusable logic belongs in modules.
- Validate config and data early: schemas, columns, types, ranges, missingness, duplicates. Report dropped rows.
- Declare dependencies and versions. Avoid hardcoded absolute paths; accept input/output paths from the caller.
- Never write secrets or sensitive data to logs or results.
- Sanity-check results: simple baselines, units, time zones, leakage, and intermediate data before final metrics.
- Long jobs must be restartable and resource-aware: bounded parallelism, streaming/chunking, checkpoints, atomic writes.
- Distinguish source-of-truth, cache, and disposable intermediates. Log enough context to debug a failed run later.

## Errors and logging

- Handle errors at the appropriate level.
- Let errors propagate when you can't meaningfully recover.
- Log errors with context before re-raising or handling.
- Log liberally with appropriate levels (DEBUG, INFO, WARNING, ERROR).
- Structure log messages with context.
- Off by default in production, easily enabled for debugging.

## How You Work on Stories/Issues

0. Use wise, practical version control practices.
1. Read the story/issue/request. Ask questions in order to achieve mental alignment with the user.
2. Study existing code patterns — follow them EXCEPT if they violate principles here, in which case discuss.
3. Understand odd existing code before removing it. Chesterton's fence applies.
4. Complete the issue with the smallest safe change.
5. Self-review for unnecessary complexity, coupling, and speculation.
6. Run tests (automated + smoke).
7. Clean up dead code and outdated documentation.
8. Commit.
