---
description: Scaffold a reproducible Yale SOM HPC research project
argument-hint: "[project name]"
---

Set up a new reproducible research project for the Yale SOM HPC cluster.

Follow the `starting-a-new-project` skill. Use the project name from `$ARGUMENTS` if given, otherwise ask me for one. Create the standard layout (code, data, slurm, logs, results), a `.gitignore` that excludes data and outputs, a short `README.md`, a `CLAUDE.md` seeded with the cluster norms (Claude Code reads `CLAUDE.md`, not `AGENTS.md`), and a minimal first test job I can submit to confirm the setup works.

Before creating anything, tell me where you plan to put the project and confirm with me.
