---
name: nchc-ssh
description: "SSH from Mac to NCHC clusters (nano4 GB200/H200, nano5 H100/H200, twaia3/Taiwan III CPU) via WSL jump host. Covers \"幫我連到 nano4/nano5/台灣杉三號/GB200/H100/t3\". Password shared across all hosts."
---

# NCHC SSH (Mac → WSL → NCHC)

Connects to NCHC clusters from Mac via the WSL jump host. Always routes through the lab machine in Taiwan for reliable access from anywhere.

## Available Clusters

| Name | GPU / Usage |
|------|-------------|
| nano4 | GB200, H200 |
| nano5 | H100, H200 |
| twaia3 (Taiwan III, t3) | CPU-bound work |

All three share the same NCHC password (`~/.nchc/pass`).

## Connection Mode (Always WSL)

Routes through the WSL machine (Mac → lab machine in Taiwan → NCHC).

| Host | Command |
|------|---------|
| nano4 | `ssh wsl "ssh nano4 '<command>'"` |
| nano5 | `ssh wsl "ssh nano5 '<command>'"` |
| twaia3 | `ssh wsl "ssh twaia3 '<command>'"` |

**Note:** Interactive aliases (`n4`, `n5`, `t3`) exist in Mac SSH config but only work for TTY sessions — use the `ssh wsl "ssh <host> 'cmd'"` form for non-interactive commands.

Direct mode (Mac → NCHC via expect scripts) is available as a fallback if the user explicitly requests it.

## Agent Workflow

### Step 1 — Identify the cluster

From the user's request:

| User says | Target |
|-----------|--------|
| nano4, GB200, H200 | nano4 |
| nano5, H100 | nano5 |
| Taiwan III, twaia3, t3, CPU job | twaia3 |

### Step 2 — Execute via WSL

```bash
ssh wsl "ssh <host> '<command>'"
```

WSL handles its own ControlMaster session caching (12h persistence), so no 2FA push is needed for subsequent connections within that window.

If the command fails, retry once before reporting failure.

## Security Rules

- Never read, display, or leak `~/.nchc/pass` (used only on the WSL side, never on Mac).
- Never commit or copy the password anywhere.

## Background

- WSL route uses `ssh wsl` as jump host; WSL-side ControlMaster manages session persistence (12h).
- Direct mode scripts at `~/.nchc/ssh-*` exist as fallback if the user requests them.
- Password file at `~/.nchc/pass` (chmod 600).
