# Agent Skills

A collection of custom agent skills for planning, development, cluster workflows, and git hygiene.

## Cluster Tools

- **cluster-info** — Provides TWCC/NCHC cluster specs, partitions, QoS limits, GPU types, MinGPU, SU billing, pricing, and architecture details.

  ```bash
  npx skills@latest add Smallfu666/skills/cluster-info
  ```

- **slurm-submission** — Use for single-GPU SLURM submission workflows for CUDA benchmarks, including sbatch scripts, Nsight Compute profiling, job arrays, and post-job review.

  ```bash
  npx skills@latest add Smallfu666/skills/slurm-submission
  ```

## Git Hygiene

- **nchc-commit** — Safe commit and push workflow for the current repository, including pre-commit inspection, credential checks, Slurm file checks, and result-folder hygiene.

  ```bash
  npx skills@latest add Smallfu666/skills/nchc-commit
  ```

## HPC Tools

- **hpc-artifact-transfer** — Pull large experiment artifacts (NCU reports, NSys, CSVs, logs, raw/encoded data) from nano4/nano5 into WSL, then pull selected files to Mac for inspection.

  ```bash
  npx skills@latest add Smallfu666/skills/hpc-artifact-transfer
  ```

## Starter Skill

- **my-skill** — A starter template for adding new custom skills to this repo.

  ```bash
  npx skills@latest add Smallfu666/skills/my-skill
  ```
