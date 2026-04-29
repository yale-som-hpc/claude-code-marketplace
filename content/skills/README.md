# Skill acceptance criteria

Every skill should:

- Have YAML frontmatter: `title`, `slug`, `description`, `related`, `updated`.
- Be directive and agent-usable: clear rules, not essay-only prose.
- Include copy-paste code examples with language-tagged fences.
- Cross-link to related skills.
- Use safe cluster defaults: no login-node compute, no credentials in scripts, no unsafe SSH key handling.
- Avoid bare shell `$SLURM_CPUS_PER_TASK`; use `${SLURM_CPUS_PER_TASK:-1}` in shell examples.
- Include a short checklist.
- Date or qualify cluster-specific facts and tell users how to verify live state.
- Avoid fixed compute-node hostnames except as explicit placeholders.

Validation checks to automate later:

- All `related:` slugs resolve.
- All local Markdown links resolve.
- All code fences have language tags.
- No lingering `TODO` markers.
- No `--ip=0.0.0.0` in tunnel examples.
- Slurm examples set thread variables unless explicitly exempted.
