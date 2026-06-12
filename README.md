# Agent Skills

A collection of custom agent skills for planning, development, cluster workflows, and git hygiene.

## Cluster Tools

- **nchc-ssh** — SSH from Mac to NCHC clusters (nano4 GB200/H200, nano5 H100/H200, Taiwan III CPU) in two modes: direct (expect scripts, push 2FA) or via WSL jump host. Asks the user which route to use.

  ```bash
  npx skills@latest add Smallfu666/skills/nchc-ssh
  ```

- **cluster-info** — Provides TWCC/NCHC cluster specs, partitions, QoS limits, GPU types, MinGPU, SU billing, pricing, and architecture details.

  ```bash
  npx skills@latest add Smallfu666/skills/cluster-info
  ```

- **slurm-submission** — General-purpose SLURM job submission on NCHC/TWCC. Enforces cluster-info prerequisites, CPU-node for CPU-only jobs, GPU opt-in, gov108018 account, handoff marker polling, and post-job seff/sacct/log review.

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

- **marker-pdf** — Convert PDF to Markdown via marker-pdf on the DGX Spark (double SSH via WSL).

  ```bash
  npx skills@latest add Smallfu666/skills/marker-pdf
  ```

- **remote-markdown-preview** — Pull remote files from any SSH host to macOS /tmp, open in Typora or default app, or copy to clipboard.

  ```bash
  npx skills@latest add Smallfu666/skills/remote-markdown-preview
  ```

## Starter Skill

- **my-skill** — A starter template for adding new custom skills to this repo.

  ```bash
  npx skills@latest add Smallfu666/skills/my-skill
  ```
