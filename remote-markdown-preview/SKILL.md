---
name: remote-markdown-preview
description: 'Use when the user wants to pull a remote Markdown/text file to macOS /tmp, open in Typora (for .md) or default app, or copy to clipboard. The user will specify a host (any SSH config alias: n4, n5, t3, wsl, dgx, lab, nano4, nano5, twaia3) and a remote path. The agent must read ~/.ssh/config to determine SSH routing. Covers phrases like "把 n4 上的 Markdown 拉回 Mac", "用 Typora 打開遠端檔案", "複製遠端第 X 到 Y 行", "遠端檔案拉到本地開", "貼到 ChatGPT", or "pull remote file".'
---

# Remote File Preview / Clipboard Skill

## Purpose

General-purpose skill for pulling remote files to macOS `/tmp`, previewing them locally, or copying content to the macOS clipboard.

- Markdown files (`.md`) should be opened in **Typora** if available.
- Other files should be opened with the system default app (`open "$LOCAL"`).
- Text/line ranges can be copied to clipboard via `pbcopy`.
- Do **not** modify the remote file.

## Host routing: how to determine the SSH command

The user may specify any host alias from `~/.ssh/config`. **The agent must read `~/.ssh/config` at runtime to decide the correct SSH route.** There are three categories:

### Category A: hosts with `RemoteCommand` (n4, n5, t3)

These aliases point to the WSL machine (`100.116.20.52`) and have `RemoteCommand ssh nano4` / `ssh nano5` / `ssh t3`. They **cannot** be used with inline commands.

**Do not use:**
```bash
ssh n4 "<command>"        # fails: "Cannot execute command-line and remote command"
```

**Instead, resolve the RemoteCommand target and use two-layer SSH:**
```bash
ssh wsl "ssh nano4 '<command>'"
ssh wsl "ssh nano5 '<command>'"
ssh wsl "ssh t3 '<command>'"
```

### Category B: hosts reachable via WSL jump (nano4, nano5, twaia3, taiwania3, dgx)

These can be reached directly from the Mac, but the known-working route is also via WSL.

**Preferred route:**
```bash
ssh wsl "ssh nano4 '<command>'"
ssh wsl "ssh nano5 '<command>'"
ssh wsl "ssh twaia3 '<command>'"
ssh wsl "ssh dgx '<command>'"
```

### Category C: hosts directly reachable from Mac (wsl, lab, lab-vpn)

Use direct SSH:
```bash
ssh wsl "<command>"
ssh lab "<command>"
ssh lab-vpn "<command>"
```

### Routing summary table

| User says | SSH config alias | Routing | Command template |
|---|---|---|---|
| n4 | `n4` → `RemoteCommand ssh nano4` | via WSL | `ssh wsl "ssh nano4 '<cmd>'"` |
| nano4 | `nano4` → `nano4.nchc.org.tw` | via WSL | `ssh wsl "ssh nano4 '<cmd>'"` |
| n5 | `n5` → `RemoteCommand ssh nano5` | via WSL | `ssh wsl "ssh nano5 '<cmd>'"` |
| nano5 | `nano5` → `nano5.nchc.org.tw` | via WSL | `ssh wsl "ssh nano5 '<cmd>'"` |
| t3 | `t3` → `RemoteCommand ssh t3` | via WSL | `ssh wsl "ssh t3 '<cmd>'"` |
| twaia3 | `twaia3` → `twnia3.nchc.org.tw` | via WSL | `ssh wsl "ssh twaia3 '<cmd>'"` |
| dgx | `dgx` → `192.168.2.60` | via WSL | `ssh wsl "ssh dgx '<cmd>'"` |
| wsl | `wsl` → `100.116.20.52` | direct | `ssh wsl "<cmd>"` |
| lab | `lab` → via Tailscale | direct | `ssh lab "<cmd>"` |
| lab-vpn | `lab-vpn` → LAN IP | direct | `ssh lab-vpn "<cmd>"` |

## Task: verify file exists

```bash
ssh <routing> "test -f '<remote-path>' && echo OK"
```

If this does not print `OK`, stop and report the failure.

## Task: pull remote file to /tmp and open

```bash
REMOTE="<remote-path>"
LOCAL="/tmp/$(basename "$REMOTE").<host>.$(date +%Y%m%d_%H%M%S)"

ssh <routing> "cat '<remote-path>'" > "$LOCAL"

if echo "$REMOTE" | grep -qi '\.md$' && open -Ra "Typora"; then
  open -a Typora "$LOCAL"
else
  open "$LOCAL"
fi

echo "$LOCAL"
```

## Task: copy entire file to clipboard

```bash
REMOTE="<remote-path>"

ssh <routing> "cat '<remote-path>'" | pbcopy
```

Optional verification:

```bash
pbpaste | head -40
```

## Task: copy line range to clipboard

```bash
REMOTE="<remote-path>"
START=<start-line>
END=<end-line>

ssh <routing> "sed -n '<START>,<END>p' '<remote-path>'" | pbcopy
```

## Task: preview lines without opening an app

```bash
REMOTE="<remote-path>"

ssh <routing> "sed -n '1,80p' '<remote-path>'"
```

## Safety rules

- Never modify the remote file.
- Never write back to the remote host.
- Never use `scp` or `rsync` unless the user explicitly asks.
- Prefer `cat`, `sed`, `head`, `tail`, `grep` for read-only inspection.
- If the file is large, do not copy the entire file to clipboard; ask for a line range first.
- If a host alias has `RemoteCommand`, detect it from `~/.ssh/config` and use the two-layer route instead.
- If a command fails, stop and report the exact command and error. Do not retry blindly.
- When in doubt about the SSH route, use the WSL two-layer form: `ssh wsl "ssh <actual-target> 'cmd'"`.

## When to use this skill

The user will say things like:

- "把 n4 上這份檔案拉到 Mac 開"
- "幫我把 n5 的 Markdown 用 Typora 打開"
- "複製 t3 上這份檔案的第 100 到 160 行"
- "把 wsl 上的 report 貼到 clipboard"
- "dgx 上那篇 paper 拉回來看"
- "遠端檔案拉到本地開"
- "pull this file from <host> and open it"

## Final response format

```
Done.
Host: <host>
Remote file: <remote path>
Local file: <local tmp path, if created>
Action: opened in Typora / opened in default app / copied to clipboard / copied line range
Notes: <only if something failed>
```
