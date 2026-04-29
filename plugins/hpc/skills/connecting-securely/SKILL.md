---
name: connecting-securely
description: SSH keys, config, agents, tunnels, and remote development without copying private keys. TRIGGER when connecting by SSH, setting up VS Code/Jupyter tunnels, forwarding ports, or troubleshooting authentication.
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

Inside tmux, SSH agent sockets can go stale. If GitHub auth worked yesterday and fails today, restart the shell or refresh `SSH_AUTH_SOCK` from the current login session.

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

On the cluster, allocate a node and start Jupyter:

```bash
srun --partition=cpunormal --cpus-per-task=4 --mem=16G --time=04:00:00 --pty bash
node=$(hostname)
echo "$node"
jupyter notebook --no-browser --port=8888 --ip=127.0.0.1
```

On your laptop, tunnel to the allocated node. Replace `c018` with the hostname printed by `hostname`:

```bash
ssh -NL 9999:localhost:8888 c018
```

Open `http://localhost:9999`.

## VS Code / Cursor

Connect to `hpc` for editing and light work. Connect to a compute node only after you have an active Slurm allocation on that node.

If Remote SSH is flaky through a jump host, add multiplexing locally:

```sshconfig
Host *
    ControlMaster auto
    ControlPath /tmp/ssh-%r@%h:%p
    ControlPersist 10m
```

You must be on the Yale network path to reach the cluster: Yale VPN/AnyConnect, campus network, or an approved bastion/jump host. If you cannot reach `hpc`, fix network access before debugging SSH keys. For long-lived tunnels, `autossh` can restart dropped tunnels, but plain `ssh -NL ...` is easier to debug first.

## Checklist

- [ ] Private keys stay on laptop or password manager, not the cluster.
- [ ] `ssh hpc` works.
- [ ] Compute-node SSH is attempted only for the node allocated to your job.
- [ ] GitHub auth works by agent forwarding or `gh`, not copied keys.
- [ ] Tunnels go to compute nodes, not login-node compute sessions.
