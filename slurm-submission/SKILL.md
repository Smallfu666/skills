---
name: slurm-submission
description: Use when submitting or reviewing SLURM jobs for CUDA benchmark workloads on TWCC/NCHC, including sbatch scripts, single-GPU Exp0/Exp1/Exp3 runs, Nsight Compute profiling, async job monitoring, job arrays, seff/sacct review, and GPU utilization checks. Do not use for partition/pricing questions (use cluster-info) or job failures/hangs (use slurm-debug).
---

# Slurm Submission

## Quick Start

Use `cluster-info` first to confirm partition, GPU type, MinGPU, wall-time limits, memory limits, architecture, and CUDA/module availability. Then keep the job as small as possible: one GPU, short wall time for smoke tests, and separate profiling from bulk sweeps.

## Core Rules

- Prefer `--gpus-per-node=1` unless the code explicitly uses multiple GPUs.
- Treat Exp0, Exp1, Exp3, and Nsight Compute as single-GPU workloads by default.
- Use job arrays for independent parameter sweeps.
- Keep benchmark logs, result CSVs, git commit hashes, and GPU model information together.
- Do not use this skill for queueing, OOM, crashes, or validation failures; use `slurm-debug` instead.

## Workflow

1. Check `cluster-info` before choosing partition or wall time.
2. Pick the template that matches the run type.
3. Fill in the benchmark command and output paths.
4. Submit with `sbatch --account=gov108018 <script>`.
5. Do not block on the job. Spawn a lightweight background worker to monitor the submission and collect completion data.
6. After the job ends, review `seff`, `sacct`, logs, and GPU utilization CSV.

## Async Monitoring

- After `sbatch` returns a job id, start a background worker agent instead of waiting inline.
- Prefer `gpt-5.4-mini` for the watcher unless the task needs heavier reasoning.
- Have the watcher poll `squeue` and `sacct`, capture the final state, and summarize logs/artifacts when the job finishes.
- Keep the foreground agent free to continue with unrelated work or the next benchmark step.

## Reference Material

See [REFERENCE.md](references/REFERENCE.md) for the canonical sbatch, Nsight Compute, sweep, and post-job review templates.
