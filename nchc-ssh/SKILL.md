---
name: nchc-ssh
description: "SSH from Mac to NCHC clusters (nano4 GB200/H200, nano5 H100/H200, twaia3/Taiwan III CPU) in two modes: direct (expect scripts, push 2FA) or via WSL jump host. Asks the user which route to use. Covers \"幫我連到 nano4/nano5/台灣杉三號/GB200/H100/t3\". Password shared across all hosts."
---

# NCHC SSH (Mac → NCHC)

Connects to NCHC clusters from Mac. Two connection modes available — the agent must ask the user which to use.

## Available Clusters

| Name | GPU / Usage |
|------|-------------|
| nano4 | GB200, H200 |
| nano5 | H100, H200 |
| twaia3 (Taiwan III, t3) | CPU-bound work |

All three share the same NCHC password (`~/.nchc/pass`).

## Connection Modes

### Mode A — Direct (Mac → NCHC)

Uses expect scripts on the Mac that automate the push-based 2FA flow.

| Host | Script |
|------|--------|
| nano4 | `~/.nchc/ssh-nano4 "<command>"` |
| nano5 | `~/.nchc/ssh-nano5 "<command>"` |
| twaia3 | `~/.nchc/ssh-twaia3 "<command>"` |

**Pros:** Lower latency, no intermediate hop.
**Cons:** Requires direct network access to `nano4.nchc.org.tw` / `twnia3.nchc.org.tw`. May not work from overseas.

### Mode B — Via WSL (Mac → WSL → NCHC)

Routes through the WSL machine (Mac → lab machine in Taiwan → NCHC).

| Host | Command |
|------|---------|
| nano4 | `ssh wsl "ssh nano4 '<command>'"` |
| nano5 | `ssh wsl "ssh nano5 '<command>'"` |
| twaia3 | `ssh wsl "ssh twaia3 '<command>'"` |

**Pros:** Works from anywhere (including overseas) since only the Mac→WSL hop needs direct access.
**Cons:** Slightly higher latency. Interactive aliases (`n4`, `n5`, `t3`) exist in Mac SSH config but only work for interactive TTY sessions, not inline commands — use the `ssh wsl "ssh <host> 'cmd'"` form for non-interactive use.

## Agent Workflow

### Step 1 — Identify the cluster

From the user's request:

| User says | Target |
|-----------|--------|
| nano4, GB200, H200 | nano4 |
| nano5, H100 | nano5 |
| Taiwan III, twaia3, t3, CPU job | twaia3 |

### Step 2 — Ask the user which route

Ask: "要直連 Mac → NCHC，還是走 WSL 跳板？"
- **直連** — use Mode A (expect scripts)
- **走 WSL** — use Mode B (ssh wsl jump)

If the user doesn't specify, default to **direct** (Mode A) since it's faster.

### Step 3 — Session check (Mode A only)

If using Mode A (direct), check if a ControlMaster session is already alive:

```bash
ssh -O check <host>
```

- **Alive:** Output is `Master running (pid=...)` on stderr, exit 0 → run immediately, no push needed.
- **Dead:** Output is `Control socket connect(...): No such file or directory` or `Connection refused` on stderr, exit 255 → inform user a push notification is needed on iPhone.

Session check is not needed for Mode B (WSL handles its own caching).

### Step 4 — Execute

**Mode A:**
```bash
~/.nchc/ssh-<host> "<command>"
```
The expect script auto-handles:
1. Sends `2` at `Login method:` prompt (Mobile APP PUSH)
2. Sends password at `Password:` prompt
3. User approves push on iPhone (Face ID)
4. Command output returned upon completion

**Mode B:**
```bash
ssh wsl "ssh <host> '<command>'"
```

## Security Rules

- Never read, display, or leak `~/.nchc/pass`.
- Never commit or copy the password anywhere.
- If using Mode A and the session is dead, always warn before triggering a push.
- If a command fails in either mode, retry once before reporting failure.

## Background

- Direct mode scripts located at `~/.nchc/ssh-*` (expect, installed at `/opt/homebrew/bin/expect`).
- Password file at `~/.nchc/pass` (chmod 600).
- SSH config has `ControlMaster auto` + `ControlPersist 12h` for all three hosts (direct mode).
- WSL route uses `ssh wsl` as jump host; WSL-side ControlMaster manages its own session persistence.
