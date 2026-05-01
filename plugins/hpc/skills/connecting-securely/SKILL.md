---
name: connecting-securely
description: Connect to the Yale SOM HPC cluster (hpc.som.yale.edu) over SSH with keys, agents, jump hosts, and tunnels — no copied private keys. TRIGGER when SSHing to the Yale SOM HPC cluster, setting up VS Code/Jupyter tunnels to a cluster compute node, forwarding ports through the cluster login node, or troubleshooting cluster SSH/agent auth.
related:
  - overview
  - managing-jobs
  - installing-software
  - using-gpus
updated: 2026-04-28
---
# Connecting Securely

Rule: use SSH keys and agent forwarding; never copy private keys onto the cluster.

## Do not copy private keys

Bad:

```bash
scp ~/.ssh/id_ed25519 hpc:~/.ssh/
```

Good:

```bash
ssh-add -l
ssh hpc
```

If you need GitHub from the cluster, use `gh auth login` or carefully scoped SSH agent forwarding. Do not store your laptop private key on GPFS. If `gh` is not installed, see [installing software](../installing-software/SKILL.md).

## Local SSH config

Example `~/.ssh/config` on your laptop:

```sshconfig
Host hpc
    HostName hpc.som.yale.edu
    User yournetid

Host b??? c???
    User yournetid
    ProxyJump hpc
    HostName %h.cm.cluster
```

Enable `ForwardAgent yes` only if you need it for Git or another SSH hop. Agent forwarding is convenient, but any process you run on the remote host can ask your agent to sign while the connection is active.

Now login works:

```bash
ssh hpc
```

Only connect to a compute node after Slurm has allocated it to you. Use the hostname printed inside your allocation, then tunnel to that node.

## Permissions

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/config
chmod 600 ~/.ssh/authorized_keys 2>/dev/null || true
chmod 700 ~
```

This is a shared system. Keep private files private: do not make your home directory, `.ssh`, `.env`, tokens, or project credentials group/world-readable.

## SSH agent checks

```bash
ssh-add -l
ssh -T git@github.com
```

Inside tmux or a long-running shell, `SSH_AUTH_SOCK` can go stale. The value is only a path to the forwarded agent socket created by your current SSH login; it is not the key itself. If GitHub auth worked yesterday and fails today, do not copy keys to the cluster. Start a fresh SSH login, restart the shell, or look into refreshing `SSH_AUTH_SOCK` so it points at a current working socket owned by you.

Principle: fix the pointer to your forwarded agent, not the location of your private key.

## GitHub from the cluster

Preferred, after installing `gh` if needed:

```bash
gh auth login
```

Alternative with explicitly enabled forwarded SSH agent:

```bash
ssh -T git@github.com
git clone git@github.com:owner/repo.git
```

## Jupyter tunnel pattern

On the cluster, allocate a small node and start Jupyter. Use the smallest allocation that fits your data, with an explicit short time limit — a tunneled Jupyter session is still an interactive job and holds those resources until you `scancel` it or the time limit hits.

```bash
srun --partition=cpunormal --cpus-per-task=2 --mem=8G --time=02:00:00 --pty bash
node=$(hostname)
echo "$node"
jupyter notebook --no-browser --port=8888 --ip=127.0.0.1
```

On your laptop, tunnel to the allocated node. Replace `c018` with the hostname printed by `hostname`:

```bash
ssh -NL 9999:localhost:8888 c018
```

Open `http://localhost:9999`. When you are done, exit the notebook and the `srun` shell. Do not leave a Jupyter session running overnight.

## VS Code / Cursor

Connect to `hpc` for editing. Connect to a compute node only after you have an active Slurm allocation on that node — never run heavy editor extensions or notebook kernels against the login node, which is shared by everyone.

You must be on a Yale network path to reach the cluster (Yale VPN/AnyConnect or campus network). Fix network access before debugging SSH keys.

## Checklist

- [ ] Private keys stay on laptop or password manager, not the cluster.
- [ ] `ssh hpc` works.
- [ ] Compute-node SSH is attempted only for the node allocated to your job.
- [ ] GitHub auth works by agent forwarding or `gh`, not copied keys.
- [ ] Tunnels go to compute nodes, not login-node compute sessions.

## Further reading

- [`ssh_config(5)`](https://man.openbsd.org/ssh_config) — every `~/.ssh/config` keyword (`ProxyJump`, `ControlMaster`, `ForwardAgent`, etc.).
- [`ssh-agent(1)`](https://man.openbsd.org/ssh-agent) — agent lifecycle, `SSH_AUTH_SOCK`, agent forwarding.
- [VS Code Remote-SSH](https://code.visualstudio.com/docs/remote/ssh) — connecting an editor to the cluster.
- [GitHub CLI auth](https://cli.github.com/manual/gh_auth_login) — `gh auth login` flow on a remote host.
