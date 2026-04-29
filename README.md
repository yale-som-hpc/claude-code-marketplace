# Yale SOM HPC — Claude Code Marketplace

A Claude Code plugin marketplace of skills for the Yale SOM HPC cluster.

## Install

In Claude Code:

```
/plugin marketplace add yale-som-hpc/claude-code-marketplace
/plugin install hpc@yale-som-hpc
```

Update later with:

```
/plugin marketplace update yale-som-hpc
```

## What's in the `hpc` plugin

Skills covering Slurm, GPUs, the GPFS filesystem, Python (uv, parallelism, acceleration), R (renv), Stata, data acquisition, and self-diagnosing wasted resources. Each skill is a directive playbook that Claude loads on demand.

| Skill | Purpose |
|---|---|
| [overview](plugins/hpc/skills/overview/SKILL.md) | Mental model, partition layout, default safety rules. |
| [connecting-securely](plugins/hpc/skills/connecting-securely/SKILL.md) | SSH keys, config, agents, Jupyter tunnels. |
| [managing-jobs](plugins/hpc/skills/managing-jobs/SKILL.md) | sbatch, arrays, dependencies, sacct/squeue. |
| [using-gpus](plugins/hpc/skills/using-gpus/SKILL.md) | When to request GPUs and how to verify they're used. |
| [using-the-filesystem](plugins/hpc/skills/using-the-filesystem/SKILL.md) | GPFS, project space, scratch, atomic writes. |
| [installing-software](plugins/hpc/skills/installing-software/SKILL.md) | Modules, uv, static binaries, Apptainer. |
| [starting-a-new-project](plugins/hpc/skills/starting-a-new-project/SKILL.md) | Reproducible project layout. |
| [running-python](plugins/hpc/skills/running-python/SKILL.md) | uv, Slurm, thread control, resumable tasks. |
| [parallel-python](plugins/hpc/skills/parallel-python/SKILL.md) | Process pools, queues, shutdown patterns. |
| [accelerating-python](plugins/hpc/skills/accelerating-python/SKILL.md) | DuckDB, Polars, Numba, when to add parallelism. |
| [running-r](plugins/hpc/skills/running-r/SKILL.md) | renv, Rscript Slurm jobs, BLAS thread control. |
| [running-stata](plugins/hpc/skills/running-stata/SKILL.md) | Batch do-files, scratch temp, license courtesy. |
| [working-with-large-data](plugins/hpc/skills/working-with-large-data/SKILL.md) | Parquet, columnar formats, query engines. |
| [acquiring-data](plugins/hpc/skills/acquiring-data/SKILL.md) | WRDS, REST APIs, scraping, request-hash caches. |
| [self-diagnosing-resource-use](plugins/hpc/skills/self-diagnosing-resource-use/SKILL.md) | sacct, seff, post-job right-sizing. |

## Repository layout

```
.claude-plugin/
└── marketplace.json          # marketplace manifest
plugins/hpc/
├── .claude-plugin/
│   └── plugin.json           # plugin manifest
└── skills/<name>/SKILL.md
```

## Skill acceptance criteria

Every skill should:

- Have YAML frontmatter: `name`, `description`, `related`, `updated`.
- Be directive and agent-usable: clear rules, not essay-only prose.
- Include copy-paste code examples with language-tagged fences.
- Cross-link related skills with relative paths.
- Use safe cluster defaults: no login-node compute, no credentials in scripts, no unsafe SSH key handling.
- Avoid bare `$SLURM_CPUS_PER_TASK` in shell; use `${SLURM_CPUS_PER_TASK:-1}`.
- Include a short checklist.
- Date or qualify cluster-specific facts and tell users how to verify live state.
- Avoid fixed compute-node hostnames except as explicit placeholders.

## Contributing

Edit `plugins/hpc/skills/<name>/SKILL.md` and open a PR. Bump `version` in `plugins/hpc/.claude-plugin/plugin.json` for releases.
