---
name: nchc-commit
description: Use when reviewing, staging, committing, or pushing changes in the current repository on NCHC or TWCC environments, especially for research code, Slurm scripts, result artifacts, or documentation. Enforces pre-commit inspection, credential checks, and clean commit hygiene.
---

# NCHC Commit Skill

Use this skill for the current repository, not a fixed path or one-off project name.

## Workflow

1. Inspect before commit:
   ```bash
   git status --short
   git diff --stat
   git diff --cached --stat
   ```
2. If any files are staged, inspect the staged diff:
   ```bash
   git diff --cached
   ```
3. Never commit sensitive access material, cookies, session files, or key material.
4. Review Slurm or `sbatch` files before committing. Private emails, environment variables, and login-node compute commands do not belong in the repo.
5. Do not commit junk such as caches, checkpoints, profiler dumps, or temporary files.
6. If result folders are committed, include context such as what ran, which job produced it, which machine or GPU was used, and which file is canonical.
7. Use precise commit messages, for example:
   ```bash
   git commit -m "exp3: add real SUM specialized baseline"
   ```
8. Push only when explicitly asked. Before pushing, inspect:
   ```bash
   git status --short
   git log --oneline -5
   ```

## Credential Check

Run the repository's standard credential check before committing staged changes.

## Rules

- Never commit `.env` files or external-service credentials.
- Slurm scripts are allowed when they do not contain sensitive material or login-node compute work.
- Do not use destructive git commands unless the user explicitly requests them.
- If sensitive material is found in staged changes, stop and rotate or revoke it first.

