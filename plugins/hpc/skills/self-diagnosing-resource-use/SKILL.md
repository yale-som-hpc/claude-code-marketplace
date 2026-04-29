---
name: self-diagnosing-resource-use
description: Diagnose whether a Yale SOM HPC cluster Slurm job used its requested CPUs, memory, GPUs, and time wisely, then right-size the next request. TRIGGER when a Slurm job on the Yale SOM HPC cluster is slow/killed/pending/idle/over-requested or needs post-run resource diagnosis.
related:
  - managing-jobs
  - using-gpus
  - using-the-filesystem
  - running-python
  - working-with-large-data
updated: 2026-04-28
---
# Self-Diagnosing Resource Use

Rule: after every serious job, check what you actually used and right-size the next job.

## Completed job accounting

```bash
sacct -j JOBID --format=JobID,JobName,Elapsed,AllocCPUS,TotalCPU,MaxRSS,State
```

For a friendlier summary:

```bash
seff JOBID
```

## Interpret CPU use

Approximate CPU efficiency:

```text
TotalCPU / (Elapsed × AllocCPUS)
```

Rules of thumb:

- >50%: reasonable for CPU-bound jobs.
- 10–50%: maybe I/O-bound or over-requested.
- <10%: probably wasteful; request fewer CPUs or parallelize correctly.

## Interpret memory use

If `MaxRSS` is 4 GB and you requested 128 GB, lower `--mem` next time. Request a safety margin, not 10× the observed peak.

## Check current jobs

```bash
squeue -u $USER -o "%.8i %.9P %.20j %.2t %.10M %.6D %.4C %.10m %R"
```

## GPU diagnosis

Inside a GPU allocation:

```bash
watch -n 1 nvidia-smi
```

Log for later:

```bash
nvidia-smi --query-gpu=timestamp,utilization.gpu,memory.used,memory.total,power.draw \
  --format=csv -l 10 > logs/gpu_${SLURM_JOB_ID}.csv &
```

Rules of thumb:

- 0 MB used: your process is not using the GPU.
- Low GPU utilization and high VRAM: model loaded but waiting on data/CPU/network.
- Sustained <10% GPU utilization: cancel and diagnose.

## Filesystem impact

```bash
du -sh /gpfs/scratch60/$USER 2>/dev/null
find /gpfs/scratch60/$USER -mtime +30 -size +1G -ls 2>/dev/null | head
find output -type f | wc -l
```

If a job creates thousands of files, strongly consider redesigning the output storage. See [using the filesystem](../using-the-filesystem/SKILL.md) for Parquet, JSONL, zip, local `/tmp`, and atomic-write patterns.

## Agent-friendly checkup script

```bash
#!/bin/bash
set -euo pipefail

jobid=${1:?usage: ./checkup.sh JOBID}

echo "=== Job accounting ==="
sacct -j "$jobid" --format=JobID,JobName,Elapsed,AllocCPUS,TotalCPU,MaxRSS,State

echo ""
echo "=== seff ==="
seff "$jobid" 2>/dev/null || true

echo ""
echo "=== Current jobs ==="
squeue -u "$USER" -o "%.8i %.9P %.20j %.2t %.10M %.6D %.4C %.10m %R"

echo ""
echo "=== Scratch usage ==="
du -sh "/gpfs/scratch60/$USER" 2>/dev/null || true
find "/gpfs/scratch60/$USER" -mtime +30 -size +1G -ls 2>/dev/null | head || true

echo ""
echo "=== GPU status, if on GPU node ==="
nvidia-smi --query-gpu=utilization.gpu,memory.used,memory.total --format=csv 2>/dev/null || true
```

## Quick rules table

| Metric | Good | Wasteful | Fix |
|---|---:|---:|---|
| CPU efficiency | >50% | <10% | request fewer CPUs or parallelize |
| Memory use | >25% requested | <5% | reduce `--mem` |
| GPU utilization | >30% training | <10% | split preprocessing, fix data loader, or use CPU |
| GPU memory | meaningful fraction | 0 MB | code is not on GPU |
| File count | <1,000/job | >10,000/job | Parquet/HDF5/zip |
| Time used | close to request | tiny fraction | request shorter time |

## Checklist

- [ ] Checked `sacct` or `seff` after the job.
- [ ] Reduced future CPU/memory/time requests if over-requested.
- [ ] Checked `nvidia-smi` for GPU jobs.
- [ ] Checked scratch usage and old files.
- [ ] Confirmed output file count is reasonable.
- [ ] Made the next job shorter or more resumable where possible.

## Further reading

- [Slurm sacct](https://slurm.schedmd.com/sacct.html) — format strings, fields like `MaxRSS`, `TotalCPU`, `Elapsed`.
- [Slurm squeue](https://slurm.schedmd.com/squeue.html) — format strings and reason codes for pending jobs.
- [`nvidia-smi` reference](https://docs.nvidia.com/deploy/nvidia-smi/) — `--query-gpu`, logging utilization, MIG.
- [py-spy](https://github.com/benfred/py-spy) — `py-spy dump --pid PID` for stuck Python processes.
