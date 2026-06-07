---
description: Set up a durable connection to the cluster that survives laptop sleep/wifi drops
---

Help me connect to the Yale SOM HPC cluster so my work survives a dropped laptop connection.

Follow the `staying-connected` and `connecting-securely` skills. Steps:

1. Ask whether I want to run Claude Code **on the login node** (Model A) or **on my laptop driving the cluster** (Model B) — explain the difference in one sentence each if I'm unsure.
2. Check my `~/.ssh/config` and add the recommended baseline (`ServerAliveInterval`, `ControlMaster`) if missing.
3. For Model A: set me up to run inside `tmux` and show me the detach/reattach commands.
4. For Model B: set up a persistent `zmx` SSH session if `zmx` is installed, otherwise show me the self-contained `ssh hpc '...'` pattern.
5. Remind me of the footprint rule: a persistent session is a shell, not a compute host — heavy work goes through `srun`/`sbatch`.

Verify which tools are actually installed (`command -v tmux zellij zmx autossh`) before recommending one.
