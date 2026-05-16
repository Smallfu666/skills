---
name: slurm-submission
description: Use when submitting or reviewing SLURM jobs for CUDA benchmark workloads on TWCC/NCHC, including sbatch scripts, single-GPU Exp0/Exp1/Exp3 runs, Nsight Compute profiling, ETA-based completion waits, post-job handoff markers, job arrays, seff/sacct review, and GPU utilization checks. Do not use for partition/pricing questions (use cluster-info) or job failures/hangs (use slurm-debug).
---

# Slurm Submission

## Quick Start

Use `cluster-info` first to confirm partition, GPU type, MinGPU, wall-time limits, memory limits, architecture, and CUDA/module availability. Then keep the job as small as possible: one GPU, short wall time for smoke tests, and separate profiling from bulk sweeps.

## Core Rules

- Prefer `--gpus-per-node=1` unless the code explicitly uses multiple GPUs.
- Treat Exp0, Exp1, Exp3, and Nsight Compute as single-GPU workloads by default.
- Use job arrays for independent parameter sweeps.
- Keep benchmark logs, result CSVs, git commit hashes, and GPU model information together.
- Do not use an agent as a long-running watcher or poll `squeue` for completion tracking.
- Prefer in-job handoff markers over polling.
- Prefer ETA-based single return checks over continuous monitoring.
- Default to marker-only completion tracking; dependent review jobs consume allocation and must be opt-in.
- Use at most one CPU-only `afterany` review job for official or expensive runs unless the user explicitly asks for separate success/failure jobs.
- Never request GPUs for review-only jobs.
- Do not auto-resubmit jobs from review scripts.
- Do not use this skill for queueing, OOM, crashes, or validation failures; use `slurm-debug` instead.

## Workflow

1. Check `cluster-info` before choosing partition or wall time.
2. Pick the template that matches the run type.
3. Fill in the benchmark command and output paths.
4. Submit with `sbatch --account=gov108018 <script>`.
5. Estimate when the job will finish, then wait once until that time plus a small buffer. Do not query `squeue` while waiting.
6. After the expected finish time, review the latest marker in `handoff/job_done/`, then inspect `seff`, `sacct`, logs, result CSVs, and GPU utilization CSV.

## ETA-Based Wait

- Estimate completion time from the submitted `--time`, the benchmark size, and prior runtime if available.
- Use the estimate to decide when to come back once, not to poll repeatedly.
- Do not turn the estimate into a watcher loop.
- If the estimate is uncertain, bias the buffer upward and rely on the handoff marker when you return.

## Completion Handoff

- Do not spawn a background watcher and do not poll `squeue` from an agent.
- For every serious benchmark job, the main sbatch script should write a completion marker when it exits:
  `handoff/job_done/job_${SLURM_JOB_ID}.json`
- Use a shell `trap` so the marker is written whether the benchmark command succeeds or fails normally.
- The marker must include:
  - `job_id`
  - `job_name`
  - `exit_status`
  - `finished_at`
  - `workdir`
  - `git_commit`
  - `stdout`
  - `stderr`
  - `gpu_util_log`
  - result CSV path or result directory
  - `next_action`
- The agent reviewing a completed job should start from the latest marker, not from `squeue`.
- For large official runs where abnormal termination must still be reviewed automatically, ask for or require opt-in to a CPU-only dependency review job:
  `sbatch --dependency=afterany:<main_job_id> --export=ALL,MAIN_JOB_ID=<main_job_id> review.sbatch`
- The review job must not request GPUs.
- AI review inside the dependency job must be a second explicit opt-in, not the default collector behavior.

## Reference Material

See [REFERENCE.md](references/REFERENCE.md) for the canonical sbatch, Nsight Compute, sweep, and post-job review templates.
