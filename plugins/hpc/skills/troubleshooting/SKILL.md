---
name: troubleshooting
description: Triage Yale SOM HPC cluster errors and map each to a fix. TRIGGER when a cluster job fails, is killed, or pends unexpectedly, or the user pastes a cluster error (GLIBC not found, no package called X, ModuleNotFoundError, command not found, bad interpreter, $'\r', nvidia-smi failure, empty Stata .out).
related:
  - managing-jobs
  - self-diagnosing-resource-use
  - installing-software
  - running-r
  - running-python
  - running-stata
  - using-gpus
updated: 2026-06-10
---
# Troubleshooting

Rule: read the actual error (and the pending **reason**), match it below, apply the fix. Don't guess from symptoms — `squeue --me -o "%T %r"`, the job's `.out`/`.log`, and `seff JOBID` almost always name the cause.

## Quick triage

| Symptom / error | Likely cause | Fix |
|---|---|---|
| Job stuck `PENDING` for ages, cluster looks idle | Targeted partition's nodes are full or reserved for a higher-priority partition (the CPU partitions overlap on shared nodes). | `squeue --me -o "%T %r"` to read the reason; `sinfo -o "%R %C"` for idle cores; resubmit to a partition with capacity (often `gpunormal` for CPU work). See [managing-jobs](../managing-jobs/SKILL.md). |
| Job **killed near a round time** (e.g. exactly 4h) | Hit the partition time limit — `default_queue` caps at 4h. | Move long work to `cpunormal`/`gpunormal` (no time limit), or split into resumable chunks. |
| `Out Of Memory` / job killed, `seff` shows high memory | `--mem` too low. | Check `seff JOBID` (MaxRSS is on the `.batch` row); set `--mem` to ~1.5–2× observed peak. See [self-diagnosing-resource-use](../self-diagnosing-resource-use/SKILL.md). |
| `./tool: ... version 'GLIBC_2.xx' not found` | Prebuilt binary needs a newer glibc than the cluster has. | Use a musl/static build, a module, or Apptainer; or compile on the cluster. See [installing-software](../installing-software/SKILL.md). |
| R: `there is no package called 'X'` | The base R module ships **no add-on packages**. | Bootstrap a project library first: `install.packages("renv")` then `renv::init()`/`renv::install(...)` on the login node. See [running-r](../running-r/SKILL.md). |
| Python: `ModuleNotFoundError` inside a job | Job ran the wrong interpreter, or env not synced. | Launch with `srun .venv/bin/python ...`; run `uv sync --frozen` on the **login node** (never inside the job). See [running-python](../running-python/SKILL.md). |
| Stata: Slurm `.out` is empty / "where's my output?" | `stata-mp -b` writes its own `.log`, not stdout. | Read the do-file's `.log` (and the stray `main.log` Stata drops in the working dir), not the `.out`. See [running-stata](../running-stata/SKILL.md). |
| `sbatch` fails: `bad interpreter`, `: not found`, or `$'\r'` | CRLF line endings (edited on Windows). | `dos2unix job.sh` or `sed -i 's/\r$//' job.sh`; set your editor to LF for `.sh`. |
| `nvidia-smi: command not found` or it errors | You're on a **login node** (no GPU driver), or no GPU was allocated. | Run inside a GPU allocation (`--gres=gpu:1`); see [using-gpus](../using-gpus/SKILL.md). |
| `git: command not found` (or other tool) | Tool isn't on the default PATH. | `module spider <tool>` then `module load`; for user tools install into `~/.local/bin`. `git` in particular needs `module load git`. |
| Python `forkserver` job hangs at startup, 0% CPU | `forkserver` deadlocks under `srun .venv/bin/python`. | Use `fork` (CPU work) or `spawn` (CUDA/threads). See [parallel-python](../parallel-python/SKILL.md). |
| `seff` reports `CPU Utilized: 00:00:00` on a job that worked | seff samples coarsely; sub-minute jobs read as 0. | Ignore CPU-efficiency on jobs under ~1–2 minutes; trust it only on longer jobs. |
| Heavy work slow / login node sluggish for everyone | Ran compute on the **login node**. | Login nodes are for editing/submitting only — put compute in `sbatch`/`srun`. See [overview](../overview/SKILL.md). |

## When nothing matches

1. `scontrol show job JOBID` — full state, including `Reason=` and resource mismatches.
2. Read the job's `.out`/`.err` and any application log to the bottom; the real error is usually the **last** traceback, not the first warning.
3. `seff JOBID` and `sacct -j JOBID --format=JobID,State,ExitCode,Elapsed,MaxRSS,ReqMem --units=G` — distinguishes OOM (killed), timeout, and clean failure.
4. Reproduce small: a 5-minute interactive job (`srun -p cpunormal -c2 --mem=8G -t 00:30:00 --pty bash`) is the fastest debug loop. Exit it the moment you're done.

## Checklist

- [ ] Read the pending **reason** (`squeue --me -o "%T %r"`) before assuming "the cluster is full."
- [ ] Read the job's `.out`/`.log` (and for Stata, the `.log` not the `.out`) to the end.
- [ ] Ran `seff`/`sacct` to tell OOM vs timeout vs clean failure apart.
- [ ] Matched the error text to a fix above rather than guessing.
