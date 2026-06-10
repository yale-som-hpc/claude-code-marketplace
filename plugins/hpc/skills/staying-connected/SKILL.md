---
name: staying-connected
description: Keep work alive across dropped SSH connections to the Yale SOM HPC cluster, and decide where Claude Code itself runs (login node vs. your laptop driving the cluster). TRIGGER when a cluster session dies on sleep/wifi drop, running Claude Code or a long agent against the cluster, keeping tmux/zmx sessions alive there, or a long-running shell losing its work on disconnect.
related:
  - connecting-securely
  - overview
  - managing-jobs
updated: 2026-06-05
---
# Staying Connected

Rule: a dropped laptop connection should never kill cluster work. Pick a model up front, make the session durable, and keep the agent's own footprint off the login node.

There are two ways to run Claude Code against the cluster. They fail differently, so they need different tools. Decide which one you are in before anything else.

| Model | Claude Code runs on | What breaks on a drop | Fix |
|---|---|---|---|
| **A** | the cluster **login node** (you SSH in, then start Claude there) | the agent process is killed when your SSH dies | session persistence on the login node (**tmux**) |
| **B** | your **laptop**, driving the cluster over SSH | each `ssh hpc <cmd>` is a fresh shell that forgets cwd/modules/env | a persistent remote shell (**zmx**) |

## Cluster facts agents should know

Current as of 2026-06-05. Verify with `command -v` on the login node before recommending a tool.

- `tmux` (3.2a) is installed system-wide on the login node. **It is the default persistence tool.**
- `mosh`, `mosh-server`, `screen`, `autossh`, and Eternal Terminal (`et`) are **not** installed on the cluster. mosh and ET need a server-side binary, so they are unavailable until SOM IT installs them — see [resilience ladder](#transport-resilience-when-your-link-is-flaky).
- `zellij` and `zmx` are not system-wide; they only work if a user installed them under `~/.local/bin`.
- `hpc.som.yale.edu` currently resolves to a single login node (`hpc-ln02`). There is a second login node on the internal network, but the public name points at one host today, so a tmux session is reliably where you left it. This can change — see the [reattach gotcha](#reattach-to-the-same-login-node).

## Baseline: SSH config (do this regardless of model)

On your laptop, `~/.ssh/config`. This keeps idle links from being silently dropped and reuses one connection for the agent's many `ssh` calls:

```sshconfig
Host hpc
    HostName hpc.som.yale.edu
    User yournetid
    ServerAliveInterval 60
    ServerAliveCountMax 3
    ControlMaster auto
    ControlPath ~/.ssh/cm-%r@%h:%p
    ControlPersist 10m
```

`ServerAliveInterval` sends a keepalive every 60s so NAT/firewalls don't time out an idle session. `ControlMaster`/`ControlPersist` multiplex repeated logins over one warm connection — this makes Model B's many remote commands fast. See [connecting securely](../connecting-securely/SKILL.md) for keys, agent forwarding, and tunnels.

## Model A — Claude Code on the login node

Run the agent inside `tmux` so it survives your laptop sleeping, your wifi dropping, or you closing the lid. The agent keeps running on the login node; you reattach later.

```bash
ssh hpc
tmux new -s work       # first time
# ...start Claude Code here, do work...
# detach with Ctrl-b then d; safe to close the laptop now
```

Reconnect later and pick up exactly where you were:

```bash
ssh hpc
tmux attach -t work    # or: tmux ls  to list sessions
```

`zellij attach -c work` is a friendlier alternative **if you have installed zellij** (it shows keybindings on screen). tmux is the safe default because it is always present.

### Keep the agent's footprint off the login node

This is the citizenship catch with Model A: the agent's own shell commands — `rg`/`find` over large trees, builds, big installs, data processing — now run **on the login node**, which is shared by everyone. A persistent tmux session is a *shell*, not a compute host.

- Quick operations (editing, `git`, small searches in a code repo, submitting jobs) are fine on the login node.
- Anything heavy — scanning multi-GB data directories, compiling, processing data, or installing large environments — belongs in `srun`/`sbatch`, including when *the agent itself* wants to run it. Tell the agent so: "do heavy work on a compute node."
- Do not leave a graveyard of detached sessions or idle agents pinning login-node memory. `tmux ls`, then `tmux kill-session -t name` when done.

### Reattach to the same login node

Today the public name lands you on one login node, so this is automatic. If the cluster ever moves to multiple public login nodes (round-robin DNS), a tmux session lives on **one** node and you must return to that node to find it. Make this drift-proof:

```bash
hostname    # run right after ssh hpc
```

If that name varies between logins, the cluster has gone multi-login: note the node your session is on and `ssh` to that specific host (e.g. `ssh hpc-ln02.som.yale.edu`) to reattach.

## Model B — Claude Code on your laptop, driving the cluster

Here the agent runs locally and reaches the cluster with `ssh hpc <cmd>`. The problem is state: each call is a fresh shell, so `cd`, `module load`, and an activated venv don't carry over. Two ways to handle it.

**Preferred: a persistent remote shell with `zmx`** (install on your laptop; it drives a durable SSH session and the agent runs commands inside it). One stateful shell on the cluster, reused across calls:

```bash
zmx run hpc -d ssh hpc       # open the remote shell once (-d goes AFTER the name)
sleep 3
zmx run hpc hostname         # later commands run INSIDE that same shell
zmx run hpc bash -lc 'cd /gpfs/project/myproject && module load Python && uv run python -V'
zmx history hpc | tail -80   # inspect what happened
```

State set in one `zmx run hpc ...` (cwd, loaded modules, env) persists to the next. See the `zmx --help` output for the authoritative command set; it changes quickly.

**Fallback without zmx:** make every remote command self-contained — re-`cd` and re-`module load` each time, since nothing persists:

```bash
ssh hpc 'cd /gpfs/project/myproject && module load Python && srun .venv/bin/python -V'
```

The `ControlMaster` baseline above makes these repeated logins fast. The footprint rule still applies: heavy work goes through `srun`/`sbatch`, not the login shell.

## Transport resilience (when your link is flaky)

tmux already protects you against a *full* disconnect (the session waits for you). These add resilience against frequent brief drops or roaming between networks. Ranked by what works on this cluster today:

1. **tmux + `ServerAliveInterval` (above).** Works now, no installs. For most people this is enough: a drop just means you `ssh hpc` and `tmux attach` again.
2. **`autossh` (laptop-side).** Auto-restarts the SSH connection when it drops, so you don't re-type `ssh hpc`. It only needs to be installed on your laptop (`brew install autossh`); nothing server-side. Pair it with tmux for persistence:
   ```bash
   autossh -M 0 -t hpc 'tmux attach -t work || tmux new -s work'
   ```
3. **`mosh` or Eternal Terminal — petition SOM IT.** Both survive sleep and network roaming far better than plain SSH (mosh over UDP; ET over TCP with real scrollback and `tmux -CC` support). Neither is installed on the cluster today. If you roam constantly, ask `somit@yale.edu` to install one. Note for mosh: it needs UDP **60000–61000** open and breaks on round-robin login DNS, so it would have to target a specific login node.

## Checklist

- [ ] You know which model you're in: Claude on the login node (A) or on your laptop (B).
- [ ] `~/.ssh/config` has `ServerAliveInterval` and `ControlMaster` set.
- [ ] Model A: the agent runs inside `tmux`, and heavy agent operations go through `srun`/`sbatch`, not the login shell.
- [ ] Model B: a persistent shell (`zmx`) holds remote state, or every command re-`cd`s and re-`module load`s.
- [ ] No idle detached sessions or idle agents left pinning the login node.

## Further reading

- [tmux manual](https://man.openbsd.org/tmux.1) — sessions, `new`/`attach`/`detach`/`ls`, prefix keys.
- [YCRC tmux guide](https://docs.ycrc.yale.edu/clusters-at-yale/guides/tmux/) — Yale's own walkthrough, including the multi-login-node reattach pattern.
- [zmx](https://zmx.sh) — persistent terminal sessions for agent + human use.
- [Mosh](https://mosh.org/) and [Eternal Terminal](https://eternalterminal.dev/) — roaming-resilient shells, if installed.
- [`ssh_config(5)`](https://man.openbsd.org/ssh_config) — `ControlMaster`, `ServerAliveInterval`, `ProxyJump`.
