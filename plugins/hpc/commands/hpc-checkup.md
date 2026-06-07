---
description: Check whether your recent Slurm jobs were right-sized, in plain language
argument-hint: "[JOBID]  (optional; defaults to your recent jobs)"
---

Run a resource checkup on the Yale SOM HPC cluster and report in plain language.

Follow the `self-diagnosing-resource-use` skill. Steps:

1. If a job ID was given (`$ARGUMENTS`), check that job. Otherwise list my recent jobs with `sacct -u $USER --starttime=today --format=JobID,JobName,Elapsed,AllocCPUS,TotalCPU,MaxRSS,State` and pick the finished ones.
2. Run `seff JOBID` on each.
3. Tell me, for each job, whether it used what it requested — CPUs, memory, time, and GPU if any — and give the concrete `--mem` / `--cpus-per-task` / `--time` I should request next time. Translate the numbers; don't just paste `seff` output.
4. Flag anything wasteful (idle GPU, <10% CPU efficiency, memory hugely over-requested) and the one change that fixes it.

Run only read-only accounting commands. Do not submit or cancel jobs.
