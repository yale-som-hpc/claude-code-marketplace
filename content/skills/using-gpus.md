---
title: Using GPUs
slug: using-gpus
description: Request GPUs only when code actively uses them, and diagnose idle allocations.
related:
  - managing-jobs
  - running-python
  - self-diagnosing-resource-use
  - using-the-filesystem
  - acquiring-data
updated: 2026-04-28
---

# Using GPUs

Rule: hold a GPU only while GPU code is actively running. Do CPU preprocessing, downloads, tokenization, and web/API calls elsewhere.

## Do you need a GPU?

Use a GPU for:

- deep learning training
- transformer/LLM inference
- CUDA/PyTorch/JAX/TensorFlow code
- RAPIDS/CuPy code that is explicitly GPU-backed

Do not use a GPU for:

- Stata regressions
- `fixest`, `pyfixest`, `statsmodels`, or ordinary CPU dataframe work
- data cleaning, merging, reshaping, tokenization
- downloading data or making network/API requests
- bootstrap/Monte Carlo unless code is actually CUDA-backed

## Request one GPU first

```bash
#!/bin/bash
#SBATCH --job-name=gpu-test
#SBATCH --partition=gpunormal
#SBATCH --gres=gpu:1
#SBATCH --time=01:00:00
#SBATCH --cpus-per-task=4
#SBATCH --mem=32G
#SBATCH --output=logs/%x_%j.out

set -euo pipefail

export OMP_NUM_THREADS=${SLURM_CPUS_PER_TASK:-1}
export MKL_NUM_THREADS=${SLURM_CPUS_PER_TASK:-1}
export OPENBLAS_NUM_THREADS=${SLURM_CPUS_PER_TASK:-1}

nvidia-smi
uv run python train.py
```

Only request multiple GPUs if the code explicitly uses multiple GPUs.

## Split preprocessing from GPU work

Bad:

```bash
#SBATCH --gres=gpu:1
uv run python download_tokenize_clean_and_train.py
```

Good:

```bash
cpu_job=$(sbatch --parsable slurm/01_preprocess_cpu.sh)
sbatch --dependency=afterok:${cpu_job} slurm/02_train_gpu.sh
```

## Monitor utilization

Inside an allocation:

```bash
watch -n 1 nvidia-smi
```

For a batch job, log utilization:

```bash
nvidia-smi --query-gpu=timestamp,utilization.gpu,utilization.memory,memory.used,power.draw \
  --format=csv -l 10 > logs/gpu_${SLURM_JOB_ID}.csv &
```

Interpretation:

- `GPU-Util` near 0% for minutes: GPU is idle.
- VRAM used but low `GPU-Util`: model loaded but waiting on CPU/data/network.
- 0 MB used: your process may not be on the GPU at all.

## Python sanity checks

```python
import torch

print(torch.cuda.is_available())
print(torch.cuda.get_device_name(0))
print(torch.cuda.memory_summary())
```

If `torch.cuda.is_available()` is false inside a GPU allocation, stop and debug before running the full job.

## Batch size

Too-small batches can starve the GPU. If GPU utilization oscillates between 0% and 100%, the data loader is likely the bottleneck.

In PyTorch, try:

```python
loader = DataLoader(dataset, batch_size=64, num_workers=4, pin_memory=True)
```

Match `num_workers` to requested CPUs. Do not set it to 32 in a 4-CPU allocation.

## See available GPUs

Check partitions and GPU types before requesting a specific GPU:

```bash
sinfo -o "%P %N %G %c %m"
sinfo --Format="partition,nodelist,gres,cpus,memory"
```

Inside a GPU allocation, check the actual card and VRAM:

```bash
nvidia-smi --query-gpu=name,memory.total --format=csv
```

Current rough guide:

- RTX 8000 / A40: 48 GB VRAM.
- A100: 40 GB or 80 GB VRAM depending on node.
- H100: 80 GB VRAM, scarcest partition.

Verify live state with `sinfo`; do not hardcode a GPU type unless you need it.

## Load CUDA when needed

Find CUDA modules:

```bash
module spider cuda
```

Then load the version your project expects, for example:

```bash
module load cuda
nvidia-smi
nvcc --version
```

If PyTorch/JAX was installed with bundled CUDA wheels, you may not need `nvcc`, but `nvidia-smi` should still work in a GPU allocation.

## Checklist

- [ ] Work actually uses CUDA/GPU libraries.
- [ ] CPU preprocessing is separate from GPU training/inference.
- [ ] `--gres=gpu:1` is used unless code is explicitly multi-GPU.
- [ ] `nvidia-smi` confirms the process uses GPU memory.
- [ ] GPU utilization is not near zero for long periods.
- [ ] Job is cancelled when interactive GPU work is done.
