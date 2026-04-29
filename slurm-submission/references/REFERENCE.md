# Slurm Submission Reference

## Scope

This skill is for CUDA benchmark submission and post-job review on TWCC/NCHC clusters.

Default assumption:

- Request 1 GPU unless the code explicitly distributes work across multiple GPUs.
- Use short, reproducible jobs for smoke tests.
- Keep Nsight Compute runs separate from bulk benchmark sweeps.
- Always record provenance in logs.
- Track completion with in-job handoff markers by default.
- Submit dependency review jobs only when the user opts in because they consume allocation.

## Required dependency

Before writing or submitting any job, use `cluster-info` first.

Do not guess:

- partition
- GPU type
- MinGPU
- wall-time limit
- memory limit
- CUDA/module availability
- node architecture

## Allocation rules

| Workload | GPU request |
| --- | --- |
| Exp0 hardware baseline | 1 GPU |
| Exp1 byte-plane bandwidth benchmark | 1 GPU |
| Exp3 progressive aggregation benchmark | 1 GPU |
| Nsight Compute profiling | 1 GPU |
| Independent parameter sweeps | Job array, 1 GPU per task |
| Multi-GPU benchmark | Only if code explicitly distributes work across GPUs |

Do not request multiple GPUs just because the node has multiple GPUs.

## Submission flow

When a job is submitted:

1. Submit the job normally with `sbatch --parsable` if the caller needs the job id.
2. Do not start a background watcher.
3. Do not poll `squeue` from an agent.
4. Let the sbatch script write `handoff/job_done/job_${SLURM_JOB_ID}.json` at exit.
5. After completion, read the newest marker and inspect the referenced logs, CSVs, GPU utilization log, `seff`, and `sacct`.

## Review job cost policy

Use this policy when deciding whether to submit a dependency review job:

| Run type | Completion path |
| --- | --- |
| Smoke test | Marker-only |
| Normal benchmark | Marker-only unless the user asks for review automation |
| Official or overnight benchmark | One CPU-only `afterany` review job if explicitly enabled |
| Failure triage | Use `slurm-debug`; do not auto-resubmit |

Rules:

- Treat every extra Slurm job as paid allocation.
- Prefer one CPU-only `afterany` review job over separate `afterok` and `afternotok` jobs.
- Keep review jobs small: 1 CPU, 4-8G memory, 5-10 minutes, no GPU.
- Make AI review a second opt-in with `RUN_AI_REVIEW=1`.
- Do not push, resubmit, or mutate benchmark inputs from a review job unless the user explicitly requested it.

## Optional dependency review wrapper

Use a wrapper only when the user explicitly wants automatic post-job review:

```bash
#!/bin/bash
set -euo pipefail
if [[ $# -lt 1 ]]; then
  echo "usage: $0 <main-sbatch-script>" >&2
  exit 2
fi
MAIN_SCRIPT="$1"
SUBMIT_REVIEW="${SUBMIT_REVIEW:-0}"
SBATCH_ACCOUNT="${SBATCH_ACCOUNT:-}"
account_args=()
if [[ -n "$SBATCH_ACCOUNT" ]]; then
  account_args=(--account="$SBATCH_ACCOUNT")
fi
main_job=$(sbatch --parsable "${account_args[@]}" "$MAIN_SCRIPT")
echo "main_job=${main_job}"
if [[ "$SUBMIT_REVIEW" == "1" ]]; then
  review_job=$(sbatch --parsable \
    "${account_args[@]}" \
    --dependency=afterany:${main_job} \
    --job-name=review_${main_job} \
    scripts/slurm_agent_review.sh "${main_job}")
  echo "review_job=${review_job}"
else
  echo "review_job=disabled"
  echo "Review handoff/job_done/job_${main_job}.json after completion."
fi
```

Submit with review enabled only when the extra CPU job is worth the cost:

```bash
SBATCH_ACCOUNT=gov108018 SUBMIT_REVIEW=1 ./submit_with_agent_review.sh exp3_job.sh
```

## CPU-only review job template

This job collects context and writes a report. It should not request GPUs.

```bash
#!/bin/bash
#SBATCH --job-name=slurm-review
#SBATCH --partition=<cpu_or_low_cost_partition>
#SBATCH --cpus-per-task=1
#SBATCH --mem=8G
#SBATCH --time=00:10:00
#SBATCH --output=logs/review_%j.out
#SBATCH --error=logs/review_%j.err
set -euo pipefail
JOB_ID="$1"
RUN_AI_REVIEW="${RUN_AI_REVIEW:-0}"
mkdir -p logs handoff/reports
REPORT="handoff/reports/review_${JOB_ID}.md"
{
  echo "# Slurm Review ${JOB_ID}"
  echo
  echo "## Accounting"
  sacct -j "$JOB_ID" --format=JobID,JobName,Partition,State,ExitCode,Elapsed,MaxRSS
  echo
  echo "## Git Status"
  git status --short || true
  echo
  echo "## Handoff Marker"
  marker="handoff/job_done/job_${JOB_ID}.json"
  if [[ -f "$marker" ]]; then
    cat "$marker"
  else
    echo "Missing marker: $marker"
  fi
  echo
  echo "## Recent Results"
  find results -type f -mmin -180 | sort | tail -100 || true
} | tee "$REPORT"
if [[ "$RUN_AI_REVIEW" == "1" ]]; then
  echo "AI review is explicitly enabled; run the project-specific agent command here."
fi
```

## Default benchmark template

```bash
#!/bin/bash
#SBATCH --job-name=byteplane-exp3
#SBATCH --partition=<partition>
#SBATCH --nodes=1
#SBATCH --gpus-per-node=1
#SBATCH --cpus-per-gpu=4
#SBATCH --mem=32G
#SBATCH --time=00:30:00
#SBATCH --output=logs/%j.out
#SBATCH --error=logs/%j.err
set -euo pipefail
mkdir -p logs results
RESULT_CSV="results/exp3_${SLURM_JOB_ID}.csv"
GPU_LOG="logs/${SLURM_JOB_ID}_gpu_util.csv"
HANDOFF_DIR="handoff/job_done"
HANDOFF_MARKER="${HANDOFF_DIR}/job_${SLURM_JOB_ID}.json"
echo "=== Job info ==="
echo "JOB_ID=${SLURM_JOB_ID}"
echo "HOST=$(hostname)"
echo "DATE=$(date -Is)"
echo "PWD=$(pwd)"
echo "=== GPU info ==="
nvidia-smi --query-gpu=index,name,memory.total,driver_version --format=csv,noheader
echo "=== CUDA info ==="
which nvcc || true
nvcc --version || true
echo "=== Git info ==="
git rev-parse HEAD || true
git status --short || true
echo "timestamp,gpu_index,util_pct,mem_used_mb,mem_total_mb" > "$GPU_LOG"
(
  while true; do
    nvidia-smi --query-gpu=index,utilization.gpu,memory.used,memory.total \
      --format=csv,noheader,nounits | tr -d ' ' \
    | awk -v ts="$(date +%Y-%m-%dT%H:%M:%S)" '{print ts","$0}' >> "$GPU_LOG"
    sleep 60
  done
) &
TRACKER_PID=$!
write_handoff_marker() {
  status=$?
  kill "$TRACKER_PID" 2>/dev/null || true
  wait "$TRACKER_PID" 2>/dev/null || true
  mkdir -p "$HANDOFF_DIR"
  git_commit="$(git rev-parse HEAD 2>/dev/null || true)"
  cat > "$HANDOFF_MARKER" <<EOF
{
  "job_id": "${SLURM_JOB_ID}",
  "job_name": "${SLURM_JOB_NAME}",
  "exit_status": ${status},
  "finished_at": "$(date -Is)",
  "workdir": "$(pwd)",
  "git_commit": "${git_commit}",
  "stdout": "logs/${SLURM_JOB_ID}.out",
  "stderr": "logs/${SLURM_JOB_ID}.err",
  "gpu_util_log": "${GPU_LOG}",
  "result_csv": "${RESULT_CSV}",
  "next_action": "Review seff, sacct, stdout, stderr, GPU utilization, and result CSV. Then write a short run report."
}
EOF
}
trap write_handoff_marker EXIT
echo "=== Run benchmark ==="
# Fill in the actual benchmark command here.
```

Submit it with:

```bash
sbatch --account=gov108018 job.sh
```

Keep `--account=gov108018` on the command line, not inside the script.

## Exp3 default command shape

Use this shape for the main throughput experiment:

```bash
./build/bench_progressive_aggregation \
  --n 100000000 \
  --segment_rows 1048576 \
  --subcolumns 8 \
  --refine_min 0 \
  --refine_max 7 \
  --load_strategy rowpack16 \
  --block 256 \
  --items_per_thread 1 \
  --validate \
  --csv "$RESULT_CSV"
```

Interpretation:

- `refinement_depth` = how many byte subcolumns are read
- `logical_bytes` = `n * (1 + refinement_depth)`
- `strategy` = `rowpack16` primary path
- `goal` = throughput vs precision depth

## Nsight Compute template

Use a separate job for profiling.

```bash
#!/bin/bash
#SBATCH --job-name=ncu-exp3
#SBATCH --partition=<partition>
#SBATCH --nodes=1
#SBATCH --gpus-per-node=1
#SBATCH --cpus-per-gpu=4
#SBATCH --mem=32G
#SBATCH --time=00:30:00
#SBATCH --output=logs/%j.out
#SBATCH --error=logs/%j.err
set -euo pipefail
mkdir -p logs results ncu
echo "JOB_ID=${SLURM_JOB_ID}"
nvidia-smi
git rev-parse HEAD || true
ncu --set full \
  --target-processes all \
  --export ncu/exp3_${SLURM_JOB_ID} \
  ./build/bench_progressive_aggregation \
    --n 100000000 \
    --segment_rows 1048576 \
    --subcolumns 8 \
    --refine_min 7 \
    --refine_max 7 \
    --load_strategy rowpack16 \
    --block 256 \
    --items_per_thread 1 \
    --validate \
    --csv results/exp3_ncu_${SLURM_JOB_ID}.csv
```

Profile one or two representative depths, not every configuration.

## Job array template

Use arrays for independent runs.

```bash
#!/bin/bash
#SBATCH --job-name=exp3-sweep
#SBATCH --partition=<partition>
#SBATCH --nodes=1
#SBATCH --gpus-per-node=1
#SBATCH --cpus-per-gpu=4
#SBATCH --mem=32G
#SBATCH --time=00:20:00
#SBATCH --array=0-7
#SBATCH --output=logs/%A_%a.out
#SBATCH --error=logs/%A_%a.err
set -euo pipefail
mkdir -p logs results
DEPTH=${SLURM_ARRAY_TASK_ID}
./build/bench_progressive_aggregation \
  --n 100000000 \
  --segment_rows 1048576 \
  --subcolumns 8 \
  --refine_min "${DEPTH}" \
  --refine_max "${DEPTH}" \
  --load_strategy rowpack16 \
  --block 256 \
  --items_per_thread 1 \
  --validate \
  --csv results/exp3_depth${DEPTH}_${SLURM_JOB_ID}.csv
```

## Post-job review

After the job ends, start from the latest handoff marker:

```bash
MARKER=$(ls -t handoff/job_done/job_*.json | head -1)
cat "$MARKER"
JOBID=$(python3 -c 'import json,sys; print(json.load(open(sys.argv[1]))["job_id"])' "$MARKER")
```

Then review Slurm accounting and utilization:

```bash
seff "$JOBID" || true
sacct -j "$JOBID" \
  --format=JobID,JobName,Partition,AllocGRES,Elapsed,CPUTime,MaxRSS,State
```

GPU utilization summary:

```bash
awk -F',' '
NR>1 {
  sum[$2]+=$3
  min[$2]=($3<min[$2] || min[$2]=="") ? $3 : min[$2]
  max[$2]=($3>max[$2]) ? $3 : max[$2]
  n[$2]++
}
END {
  for (g in sum)
    printf "GPU %s: avg=%.1f%% min=%.1f%% max=%.1f%% samples=%d\n", g, sum[g]/n[g], min[g], max[g], n[g]
}' logs/${JOBID}_gpu_util.csv
```

Interpretation:

| Observation | Meaning | Next action |
| --- | --- | --- |
| GPU util high, CSV valid | Good benchmark run | Keep config |
| GPU util low but runtime short | Normal for tiny smoke test | Increase `n` |
| GPU util low in long run | Workload sizing or CPU bottleneck | Inspect kernel launch or data path |
| One GPU used, others idle | Over-requested GPUs | Request 1 GPU next time |
| Wall time far below request | Time estimate too loose | Reduce `--time` |
| Validation failed | Not a SLURM issue | Use `slurm-debug` or code review |

## Smoke vs real benchmark

Smoke test:

- small `n`
- short wall time
- validate correctness
- not used for final performance claims

Real benchmark:

- `n = 100,000,000` or larger
- stable repeated timings
- CSV archived
- logs archived
- git commit recorded
- GPU type recorded

Do not cite smoke-test throughput as evidence.

## Required artifacts

Each serious run should leave:

- `logs/<jobid>.out`
- `logs/<jobid>.err`
- `logs/<jobid>_gpu_util.csv`
- `results/*.csv`
- git commit hash in log
- GPU model in log
- exact command in log

If any of these are missing, the result is weaker.

## When not to use this skill

Use `cluster-info` for:

- partition list
- GPU type
- QoS
- pricing
- account limits

Use `slurm-debug` for:

- job stuck in queue
- CUDA crash
- missing module
- OOM
- segmentation fault
- validation failure
- Nsight failure
- no output generated
