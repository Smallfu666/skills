# Agent Skills

A collection of custom agent skills for planning, development, and cluster workflows.

## Cluster Tools

- **cluster-info** — Provides TWCC/NCHC cluster specs, partitions, QoS limits, GPU types, MinGPU, SU billing, pricing, and architecture details.

  ```bash
  npx skills@latest add Smallfu666/skills/cluster-info
  ```

- **slurm-submission** — Use for single-GPU SLURM submission workflows for CUDA benchmarks, including sbatch scripts, Nsight Compute profiling, job arrays, and post-job review.

  ```bash
  npx skills@latest add Smallfu666/skills/slurm-submission
  ```

## Starter Skill

- **my-skill** — A starter template for adding new custom skills to this repo.

  ```bash
  npx skills@latest add Smallfu666/skills/my-skill
  ```
