---
name: slurm-submission
description: Use when submitting or reviewing SLURM jobs on NCHC/TWCC, including sbatch script creation, completion polling via handoff markers, and post-job seff/sacct/log review. Do not use for partition/pricing questions (use cluster-info) or job failures/hangs (use slurm-debug).
---

# Slurm Submission

## Prerequisite: cluster-info

Before writing or submitting any job, use `cluster-info` first. Do not guess partition, GPU type, wall-time limit, memory limit, or node architecture.

## Core Rules

- Use `--account=gov108018` on the sbatch command line, not inside the script.
- **GPU is opt-in.** Default to CPU-only. Ask the user before adding `--gpus-per-node`.
- **CPU jobs must use CPU partitions** (e.g., NGX8G, NGX32G). Do not request a GPU node for CPU-only work.
- nvidia-smi GPU utilization logging is opt-in. Only add it when the user explicitly asks or when GPU profiling is the goal of the job.
- Use a completion handoff marker in `handoff/job_done/` for tracking job completion. Poll the marker file rather than `squeue`.
- Keep polling until the marker appears. Agent decides the polling interval based on wall-time and job type.
- Once the marker is found, run post-job review (seff, sacct, logs). Do not stop early.
- If the marker is still absent after a reasonable timeout relative to wall-time, run one lightweight `sacct` check to report the job state, then stop.
- The polling agent must not submit new jobs, mutate inputs, push git, or auto-resubmit.
- Do not use this skill for stuck jobs, OOM, crashes, or validation failures; use `slurm-debug` instead.

## Workflow

1. **Check `cluster-info`** — confirm partition, GPU type, wall-time limit, memory limit, and node architecture.
2. **Ask the user** what they want to run and whether it needs a GPU.
3. **Write the sbatch script** — create a script matching their workload. Include a `trap`-based handoff marker.
4. **Submit** with `sbatch --account=gov108018 <script>`.
5. **Poll for completion** — check `handoff/job_done/` for the marker file. Agent determines polling frequency.
6. **Once the marker appears**, read it and run post-job review: inspect `seff`, `sacct`, logs, and any output files.

## Completion Handoff

Every sbatch script should write a completion marker when it exits:

```
handoff/job_done/job_${SLURM_JOB_ID}.json
```

Use a shell `trap` so the marker is written whether the job succeeds or fails normally. The marker should include:

- `job_id`, `job_name`, `exit_status`, `finished_at`, `workdir`
- `stdout`, `stderr` paths
- `next_action` — e.g. "Review seff, sacct, stdout, stderr, and output files."

The agent reviewing a completed job should start from the latest marker, not from `squeue`.

## Post-Job Review

After a job completes, inspect:

- `seff <job_id>`
- `sacct -j <job_id> --format=JobID,JobName,Partition,AllocGRES,Elapsed,CPUTime,MaxRSS,State`
- stdout / stderr logs
- Any result files the job produced
- GPU utilization log if one was generated
