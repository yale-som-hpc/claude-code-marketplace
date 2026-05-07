<div align="center">
  <img src="assets/yale-som-claude.svg" alt="Yale SOM × Claude" width="200" />
  <h1>Yale SOM HPC<br>Claude Code Marketplace</h1>

</div>

This repo is a [Claude Code](https://code.claude.com/docs/en/overview) plugin marketplace of skills for the Yale SOM HPC cluster (`hpc.som.yale.edu`). Install it into Claude Code to help you get work done on the HPC — write great code, submit jobs, request a GPU, set up a project, query WRDS, diagnose problems, etc.

The skills were developed with two goals in mind.

1. **Be skillful.** Right-sized jobs start sooner. The right tool runs an order of magnitude faster. Atomic writes don't lose your weekend's work. Skill is mostly selfish — your research, your time, your results.
2. **Be polite.** The HPC is a shared resource. Other people are running jobs right now on the same nodes, GPUs, and GPFS metadata servers.

## What's in the `hpc` plugin

Start with [overview](plugins/hpc/skills/overview/SKILL.md) — it's the front door, with the mental model, partition layout, and how the rest fit together. Then:

### Connect & set up

| Skill | Purpose |
|---|---|
| [connecting-securely](plugins/hpc/skills/connecting-securely/SKILL.md) | SSH keys, config, agents, Jupyter tunnels. |
| [starting-a-new-project](plugins/hpc/skills/starting-a-new-project/SKILL.md) | Reproducible project layout. |
| [installing-software](plugins/hpc/skills/installing-software/SKILL.md) | Modules, uv, static binaries, Apptainer. |

### Run jobs

| Skill | Purpose |
|---|---|
| [managing-jobs](plugins/hpc/skills/managing-jobs/SKILL.md) | sbatch, arrays, dependencies, right-sizing loop. |
| [using-gpus](plugins/hpc/skills/using-gpus/SKILL.md) | When to request GPUs and how to verify they're used. |
| [using-the-filesystem](plugins/hpc/skills/using-the-filesystem/SKILL.md) | GPFS, project space, scratch, atomic writes. |
| [running-python](plugins/hpc/skills/running-python/SKILL.md) | uv, Slurm, thread control, resumable tasks. |
| [parallel-python](plugins/hpc/skills/parallel-python/SKILL.md) | Worker sizing, spawn-vs-fork, nested-parallelism warning. |
| [accelerating-python](plugins/hpc/skills/accelerating-python/SKILL.md) | DuckDB, Polars, Numba, when to add parallelism. |
| [running-r](plugins/hpc/skills/running-r/SKILL.md) | renv, Rscript Slurm jobs, BLAS thread control. |
| [running-stata](plugins/hpc/skills/running-stata/SKILL.md) | Batch do-files, scratch temp, license courtesy. |

### Work with data

| Skill | Purpose |
|---|---|
| [working-with-large-data](plugins/hpc/skills/working-with-large-data/SKILL.md) | Parquet, columnar formats, query engines. |
| [acquiring-data](plugins/hpc/skills/acquiring-data/SKILL.md) | WRDS, REST APIs, scraping, connection pooling, request-hash caches. |

### Diagnose

| Skill | Purpose |
|---|---|
| [self-diagnosing-resource-use](plugins/hpc/skills/self-diagnosing-resource-use/SKILL.md) | sacct, seff, post-job right-sizing. |

## Install

You need [Claude Code](https://code.claude.com/docs/en/overview) installed. This repo is a [plugin marketplace](https://code.claude.com/docs/en/discover-plugins) — a catalog Claude Code can browse — and `hpc` is the [plugin](https://code.claude.com/docs/en/plugins) inside it that bundles all the skills. To use the skills, you add the marketplace once, then install the plugin from it. Nothing runs on the cluster until you ask Claude to do something there.

**TL;DR** — in any Claude Code session, run:

```
/plugin marketplace add yale-som-hpc/claude-code-marketplace
/plugin install hpc@yale-som-hpc
```

If you'd rather use a GUI, pick the surface below that matches how you use Claude Code. Plugin state is shared per-user, so installing once enables `hpc` in every local Claude Code session on your machine.

### Claude Code Desktop app

The Desktop app has a graphical plugin browser. In your session, click the **+** button next to the prompt box and choose **Plugins → Add plugin**. Add this marketplace:

```
yale-som-hpc/claude-code-marketplace
```

Then install `hpc` from the browser. The CLI commands below also work — type them straight into the prompt box. See the [Desktop plugins docs](https://code.claude.com/docs/en/desktop#install-plugins) for the full UI walkthrough. Don't have the app yet? [Download Claude Desktop](https://claude.com/download).

### Claude Code in VS Code or Cursor

In the Claude Code prompt box, type:

```
/plugin
```

That opens the plugin manager. Go to the **Marketplaces** tab, click **Add marketplace**, and paste:

```
yale-som-hpc/claude-code-marketplace
```

Then switch to the **Discover** tab and install `hpc`. If you prefer typing, the CLI commands below work inside the chat box too.

### Claude Code CLI (terminal)

Inside Claude Code:

```
/plugin marketplace add yale-som-hpc/claude-code-marketplace
/plugin install hpc@yale-som-hpc
```

Update later with `/plugin marketplace update yale-som-hpc`. The full command list is in the [plugins reference](https://code.claude.com/docs/en/plugins-reference).

### Per-project (committed to a repo)

If you want every collaborator on a project to get the plugin automatically, commit the marketplace reference into the repo. Add a `.claude/settings.json` like:

```json
{
  "extraKnownMarketplaces": {
    "yale-som-hpc": {
      "source": {
        "source": "github",
        "repo": "yale-som-hpc/claude-code-marketplace"
      }
    }
  },
  "enabledPlugins": {
    "hpc@yale-som-hpc": true
  }
}
```

Anyone who opens the project in a local Claude Code session (Desktop, VS Code, Cursor, CLI) will be prompted to install the marketplace and plugin the first time they trust the folder. See [Configure team marketplaces](https://code.claude.com/docs/en/discover-plugins#configure-team-marketplaces) for the full reference.

## Verify

Inside Claude Code, run `/plugin`. You should see `hpc@yale-som-hpc` listed and enabled. Ask Claude something cluster-shaped ("write me an sbatch script for a Polars job on the SOM HPC cluster") and it should pull from the relevant skill.

## Repository layout

```
.claude-plugin/
└── marketplace.json          # marketplace manifest
plugins/hpc/
├── .claude-plugin/
│   └── plugin.json           # plugin manifest (bump version on release)
└── skills/<name>/SKILL.md    # one directory per skill
```

A skill is a single `SKILL.md` with YAML frontmatter and a directive playbook body. Claude Code's skill loader reads the frontmatter to decide when to load each skill into context.

## Contributing

Most skills are short — a fix or new section is usually a one-PR change. If you're not ready to write a fix, file an issue.

### Filing an issue

Open an issue at <https://github.com/yale-som-hpc/claude-code-marketplace/issues>. Useful issues fall into a few buckets:

- **Skill is wrong.** A code example that fails on the current cluster, a partition name that no longer exists, a `module load` that errors. Include the command you ran and the error you got. Cluster-truth bugs are the highest-priority fixes.
- **Skill is missing.** A pattern you keep re-deriving (e.g. "how do I run a long-running Stan job", "how do I share a conda-installed binary with collaborators"). Describe the workflow and roughly what the rule should be — we'll figure out where it fits.
- **Skill is misfiring.** Claude is loading a skill on your laptop where it shouldn't, or not loading it on the cluster where it should. Include the prompt that triggered it and which skill fired.
- **Doc gap.** A README, install, or contribution-flow problem. Just describe what you tried and what was unclear.

You don't need to propose a fix. A clear "this happened, I expected this, instead I got this" is enough.

### Workflow

1. Fork or branch.
2. Edit the relevant `plugins/hpc/skills/<name>/SKILL.md`. For a new skill, copy an existing one as a template.
3. Bump the `version` in `plugins/hpc/.claude-plugin/plugin.json` (semver: bug fix → patch, content addition → minor, breaking reorganization → major).
4. Open a PR. Describe what changed and why. If you verified anything against the running cluster, say so.

### Skill acceptance criteria

Every skill should:

- Have YAML frontmatter: `name`, `description`, `related`, `updated`.
- Have a `description` that gates on Yale SOM HPC cluster context, with a `TRIGGER when ...` clause naming concrete cluster signals (Slurm, GPFS, sbatch, login node, etc.). This stops the skill from auto-firing on someone's laptop.
- Be directive and agent-usable: rules, not essay prose.
- Include copy-paste code examples with language-tagged fences.
- Cross-link related skills with relative paths.
- Use safe cluster defaults: no login-node compute, no credentials in scripts, no unsafe SSH key handling, `${SLURM_CPUS_PER_TASK:-1}` (never bare).
- End with a short checklist and a `## Further reading` section linking to canonical external docs.
- Date cluster-specific facts (`updated:` frontmatter) and tell users how to verify live state (`sinfo -s`, `module spider`, etc.).
- Avoid fixed compute-node hostnames except as explicit placeholders.

### What does not belong in these skills

- Generic Python/R/Stata style or framework tutorials. If a rule applies on a laptop the same way it applies on the cluster, it does not belong here.
- Yale-specific operational policy that changes (which mailing list to email, who the current admin is, how to request an account). Link to the policy page instead.
- Long narrative explanations. Agents work best from short directive rules with a clear "why" sentence.

### Verifying changes against the cluster

If you change an sbatch template, a `module load` line, a partition name, or a path, run it on the cluster before merging. The skills exist so that an agent following them produces something that actually works; drift between the skills and live cluster state is the main failure mode.

## License

Released into the public domain under [The Unlicense](LICENSE).
