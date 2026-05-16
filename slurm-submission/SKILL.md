---
name: slurm-submission
description: Use when submitting or reviewing SLURM jobs for CUDA benchmark workloads on TWCC/NCHC, including sbatch scripts, single-GPU Exp0/Exp1/Exp3 runs, Nsight Compute profiling, 3-5 minute completion polling, post-job handoff markers, job arrays, seff/sacct review, and GPU utilization checks. Do not use for partition/pricing questions (use cluster-info) or job failures/hangs (use slurm-debug).
---

# Slurm Submission

## Quick Start

Use `cluster-info` first to confirm partition, GPU type, MinGPU, wall-time limits, memory limits, architecture, and CUDA/module availability. Then keep the job as small as possible: one GPU, short wall time for smoke tests, and separate profiling from bulk sweeps.

## Core Rules

- Prefer `--gpus-per-node=1` unless the code explicitly uses multiple GPUs.
- Treat Exp0, Exp1, Exp3, and Nsight Compute as single-GPU workloads by default.
- Use job arrays for independent parameter sweeps.
- Keep benchmark logs, result CSVs, git commit hashes, and GPU model information together.
- Poll every 3-5 minutes checking for the handoff marker in `handoff/job_done/`. Do not poll `squeue` — check the marker file instead.
- Prefer in-job handoff markers over polling `squeue`.
- The agent should keep polling until the marker appears, then run the full post-job review. Do not stop early.
- If the marker is still absent after a reasonable timeout, run one lightweight `sacct` (preferred) or `squeue` check to report the job state.
- The polling agent must not submit new jobs, mutate benchmark inputs, push git, or auto-resubmit.
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
5. Poll every 3-5 minutes checking `handoff/job_done/` for the marker file. Do not poll `squeue`.
6. Once the marker appears, read it and inspect `seff`, `sacct`, logs, result CSVs, and GPU utilization CSV.

## Completion Polling

- Poll every 3-5 minutes by checking for the marker file in `handoff/job_done/`. Check the file, not `squeue`.
- Keep polling until the marker appears. Do not stop early.
- If the marker is found, run the full post-job review.
- If the marker is absent after a reasonable timeout, run one lightweight `sacct` (preferred) or `squeue` check to report the job state.

## Completion Handoff

- Poll the marker file in `handoff/job_done/` every 3-5 minutes. Do not poll `squeue`.
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

## Completion Polling

Since benchmark jobs complete within 1 hour, poll every 5 minutes instead of estimating ETA:

1. Submit the job.
2. Every 5 minutes, check `handoff/job_done/` for the marker file (check file, not `squeue`).
3. If the marker exists, read it and run the post-job review (see Workflow step 6).
4. If the marker is absent, sleep 5 minutes and recheck.
5. Repeat up to 1 hour (~12 checks).
6. If still absent after 1 hour, run one lightweight `sacct` (preferred) or `squeue` check to report the job state, then stop.
7. The polling agent must not submit new jobs, mutate benchmark inputs, push git, or auto-resubmit.

Example delegation prompt:

> "Check handoff/job_done/ every 5 minutes for up to 1 hour. If marker found, run post-job review (seff, sacct, GPU util, logs). If absent after 1 hour, run one sacct to report state, then stop. Do not submit jobs, mutate inputs, push git, or auto-resubmit."

## Reference Material

See [REFERENCE.md](references/REFERENCE.md) for the canonical sbatch, Nsight Compute, sweep, and post-job review templates.
