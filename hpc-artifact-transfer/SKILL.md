---
name: hpc-artifact-transfer
description: Use when the user needs to move large experiment artifacts from nano4/nano5 to WSL, and optionally from WSL to Mac, without committing large files to Git

---

# HPC Artifact Transfer

## Overview

Move experiment artifacts from an HPC machine (`nano4` or `nano5`) into the user's WSL artifact hub, then provide a Mac-side command to pull selected files for local inspection.

This skill is for large or non-Git-friendly artifacts:

- `.ncu-rep`
- `.nsys-rep`
- `.qdrep`
- Nsight output directories
- large `.csv`
- Slurm logs
- benchmark result directories
- raw datasets
- encoded artifacts
- large intermediate outputs

**Core principle:** HPC machines compute. WSL stores heavy artifacts. Mac only pulls what the user wants to inspect locally. GitHub stores code, specs, docs, small summaries, plots, and manifests.

**Announce at start:** "I'm using the hpc-artifact-transfer skill to pull experiment artifacts into WSL."

---

## Topology

Expected data flow:

```text
nano4 / nano5
    |
    | WSL pulls via rsync
    v
WSL artifact hub
    |
    | Mac pulls via rsync when needed
    v
Mac local viewer / analysis tools
```

Do **not** require `nano4` or `nano5` to push files back to WSL. Prefer pull-based transfer from WSL because it is simpler, repeatable, and avoids inbound networking problems.

---

## Required User Inputs

Ask for exactly these two inputs if they are not already known:

1. Which HPC machine?
   - `nano4`
   - `nano5`

2. What remote path should be pulled?
   - A file or directory path on the selected machine.
   - If the user does not know the path, ask them to query the HPC-side agent using the prompt in **Step 1a**.

Optional inputs, only ask if needed:

- Artifact kind: `ncu`, `nsys`, `csv`, `logs`, `raw`, `encoded`, `runs`, `misc`
- Human-readable name for this transfer

If the artifact kind or name is not provided, infer them from the remote path and file extensions.

---

## Step 0: Confirm Execution Context

This skill should usually run on WSL.

Check:

```bash
uname -a
pwd
```

If running on Mac, do **not** pull directly from `nano4`/`nano5` unless the user explicitly wants that. Instead, provide a command that SSHes into WSL and runs the WSL-side pull command.

If running on `nano4` or `nano5`, do **not** push artifacts outward. Report the path and instruct the WSL-side agent to pull it.

---

## Step 1: Get the Remote Artifact Path

### Step 1a: Prompt for the HPC-side Agent

If the user does not know the remote path, give them this prompt to paste into the `nano4` or `nano5` agent:

```markdown
I need to transfer experiment artifacts back to my WSL artifact hub.

Please identify the exact path I should pull from this machine.

Return only this structured block:

```text
host=<nano4-or-nano5>
artifact_kind=<ncu|nsys|csv|logs|raw|encoded|runs|misc>
recommended_name=<short-descriptive-name>
remote_path=<absolute-path-to-file-or-directory>
size=<du -sh result>
important_files=<short list of key files, especially .ncu-rep if present>
notes=<anything important about whether this is complete or partial>
```

Before answering, verify the path exists with:

```bash
test -e <path> && du -sh <path>
find <path> -maxdepth 2 -type f | head -50
```
```

### Step 1b: Validate the Remote Path from WSL

Once `host` and `remote_path` are known, validate from WSL:

```bash
HOST="<nano4-or-nano5>"
REMOTE_PATH="<absolute-remote-path>"

ssh "$HOST" "test -e '$REMOTE_PATH' && du -sh '$REMOTE_PATH' && find '$REMOTE_PATH' -maxdepth 2 -type f | head -50"
```

If validation fails:

- Do not guess.
- Ask the HPC-side agent to re-check the path.
- Common causes:
  - Path is on a compute node local disk and not visible from login node.
  - Job wrote to `$SLURM_TMPDIR` and files were not copied out.
  - The path has a typo.
  - The host alias is wrong.
  - Permissions are missing.

---

## Step 2: Classify Artifact Kind

Infer the artifact kind using this priority:

| Condition                                                    | kind      |
| ------------------------------------------------------------ | --------- |
| Contains `.ncu-rep`                                          | `ncu`     |
| Contains `.nsys-rep`, `.qdrep`, `.sqlite` from Nsight Systems | `nsys`    |
| Mostly `.csv` and result tables                              | `csv`     |
| Slurm `.out`, `.err`, `.log` files                           | `logs`    |
| Raw input data such as `.f64le.bin`                          | `raw`     |
| Encoded artifacts such as `.buff64`, `plane_###.bin`, `manifest.json`, `segment_meta.csv` | `encoded` |
| Full experiment result directory                             | `runs`    |
| Unknown mixed output                                         | `misc`    |

If uncertain, choose `runs`. Do not overfit. The directory name and manifest will preserve context.

---

## Step 3: Choose WSL Destination

Use this root:

```bash
ARTIFACT_ROOT="$HOME/artifacts/hpc"
```

Store by host and kind:

```text
~/artifacts/hpc/
  nano4/
    ncu/
    nsys/
    csv/
    logs/
    raw/
    encoded/
    runs/
    misc/
  nano5/
    ncu/
    nsys/
    csv/
    logs/
    raw/
    encoded/
    runs/
    misc/
```

Destination naming format:

```text
YYYY-MM-DD__recommended_name
```

Example:

```text
~/artifacts/hpc/nano4/ncu/2026-05-05__rmsnorm_layernorm_v2_job38967/
```

If no recommended name exists, derive one from `basename "$REMOTE_PATH"`.

Sanitize names:

```bash
SAFE_NAME="$(echo "$NAME" | tr ' /:' '___' | tr -cd 'A-Za-z0-9._+-')"
```

---

## Step 4: Pull from HPC to WSL

Use `rsync`, not `scp`, for normal transfers.

### Directory transfer

```bash
HOST="<nano4-or-nano5>"
REMOTE_PATH="<absolute-remote-directory>"
KIND="<ncu|nsys|csv|logs|raw|encoded|runs|misc>"
NAME="<short-descriptive-name>"

DATE="$(date +%F)"
SAFE_NAME="$(echo "$NAME" | tr ' /:' '___' | tr -cd 'A-Za-z0-9._+-')"
DEST="$HOME/artifacts/hpc/$HOST/$KIND/${DATE}__${SAFE_NAME}"

mkdir -p "$DEST"

rsync -avP --partial --append-verify \
  "$HOST:${REMOTE_PATH%/}/" \
  "$DEST/"
```

### Single file transfer

If the remote path is a single file:

```bash
HOST="<nano4-or-nano5>"
REMOTE_PATH="<absolute-remote-file>"
KIND="<ncu|nsys|csv|logs|raw|encoded|runs|misc>"
NAME="<short-descriptive-name>"

DATE="$(date +%F)"
SAFE_NAME="$(echo "$NAME" | tr ' /:' '___' | tr -cd 'A-Za-z0-9._+-')"
DEST="$HOME/artifacts/hpc/$HOST/$KIND/${DATE}__${SAFE_NAME}"

mkdir -p "$DEST"

rsync -avP --partial --append-verify \
  "$HOST:$REMOTE_PATH" \
  "$DEST/"
```

### Large transfer option

For very large raw or encoded artifacts, add a bandwidth limit only if the user asks or the connection is unstable:

```bash
--bwlimit=50000
```

This means roughly 50 MB/s.

---

## Step 5: Write Transfer Manifest

After every transfer, create:

```bash
MANIFEST="$DEST/transfer_manifest.txt"

{
  echo "source_host=$HOST"
  echo "source_path=$REMOTE_PATH"
  echo "dest_host=wsl"
  echo "dest_path=$DEST"
  echo "artifact_kind=$KIND"
  echo "name=$SAFE_NAME"
  echo "pulled_at=$(date -Is)"
  echo "transfer_tool=rsync"
} > "$MANIFEST"
```

Also create a compact file listing:

```bash
find "$DEST" -maxdepth 3 -type f | sort > "$DEST/file_list.txt"
du -sh "$DEST" > "$DEST/size.txt"
```

---

## Step 6: Verify Transfer

Run:

```bash
du -sh "$DEST"
find "$DEST" -maxdepth 2 -type f | head -50
```

For NCU reports:

```bash
find "$DEST" -name "*.ncu-rep" | sort
```

For CSV outputs:

```bash
find "$DEST" -name "*.csv" | sort
```

For logs:

```bash
find "$DEST" \( -name "*.out" -o -name "*.err" -o -name "*.log" -o -name "*.txt" \) | sort | head -50
```

If expected files are missing:

- Do not declare success.
- Report what was transferred.
- Ask the HPC-side agent to verify whether the job completed and whether files were written elsewhere.

---

## Step 7: Provide Mac Pull Command

After successful WSL transfer, provide a Mac command.

### General Mac pull

```bash
mkdir -p "$HOME/Downloads/hpc-artifacts/<name>"

rsync -avP \
  "wsl:<wsl-dest-path>/" \
  "$HOME/Downloads/hpc-artifacts/<name>/"
```

### NCU-specific Mac pull

For `.ncu-rep`, prefer:

```bash
mkdir -p "$HOME/Downloads/ncu/<name>"

rsync -avP \
  "wsl:<wsl-dest-path>/" \
  "$HOME/Downloads/ncu/<name>/"

find "$HOME/Downloads/ncu/<name>" -name "*.ncu-rep" | sort
```

Example:

```bash
mkdir -p "$HOME/Downloads/ncu/rmsnorm_layernorm_v2_job38967"

rsync -avP \
  "wsl:~/artifacts/hpc/nano4/ncu/2026-05-05__rmsnorm_layernorm_v2_job38967/" \
  "$HOME/Downloads/ncu/rmsnorm_layernorm_v2_job38967/"

find "$HOME/Downloads/ncu/rmsnorm_layernorm_v2_job38967" -name "*.ncu-rep" | sort
```

### If the user has a custom alias such as `shwsl`

If `shwsl` is the user's SSH alias for WSL from Mac, replace `wsl:` with `shwsl:`:

```bash
rsync -avP \
  "shwsl:<wsl-dest-path>/" \
  "$HOME/Downloads/ncu/<name>/"
```

Do not assume the alias. Use `wsl` unless the user explicitly says the alias is `shwsl`.

---

## Step 8: Optional NAS Mirror

Only mirror to NAS if the user asks or the artifact is important enough for cold storage.

Example pattern:

```bash
NAS_DEST="/path/to/nas/gpu-byteplane/$HOST/$KIND/${DATE}__${SAFE_NAME}"

mkdir -p "$NAS_DEST"

rsync -avP --partial --append-verify \
  "$DEST/" \
  "$NAS_DEST/"
```

After mirroring:

```bash
du -sh "$NAS_DEST"
find "$NAS_DEST" -maxdepth 2 -type f | head -50
```

Do not delete the WSL copy after NAS mirror unless the user explicitly asks.

---

## Step 9: Git Policy Check

If the artifact was originally inside a Git repo, verify it is not tracked.

Run in the repo:

```bash
git status --short --ignored
git ls-files | grep -E '\.(ncu-rep|nsys-rep|qdrep|sqlite|f64le\.bin|buff64)$' || true
```

If heavy artifacts are tracked:

```bash
git rm --cached <path>
```

Then update `.gitignore`:

```gitignore
# Heavy profiler artifacts
*.ncu-rep
*.nsys-rep
*.qdrep
*.sqlite

# Large data artifacts
*.f64le.bin
*.buff64

# Local artifact staging
artifacts/
large_results/
ncu_reports/
nsys_reports/
```

Do not use `git add .` in repositories that may contain artifacts. Add explicit paths only.

---

## Reusable Helper Script

If the helper script does not exist on WSL, create it at:

```bash
~/bin/hpc-pull-artifact
```

Script:

```bash
#!/usr/bin/env bash
set -euo pipefail

usage() {
  cat <<'USAGE'
usage:
  hpc-pull-artifact <host> <remote_path> <kind> [name]

host:
  nano4 | nano5

kind:
  ncu | nsys | csv | logs | raw | encoded | runs | misc

examples:
  hpc-pull-artifact nano4 /work/$USER/project/ncu_reports ncu rmsnorm_v2_job38967
  hpc-pull-artifact nano5 /work/$USER/project/results/exp4/run_xxx runs exp4_run_xxx
USAGE
  exit 1
}

if [ "$#" -lt 3 ] || [ "$#" -gt 4 ]; then
  usage
fi

HOST="$1"
REMOTE_PATH="$2"
KIND="$3"
NAME="${4:-$(basename "$REMOTE_PATH")}"

case "$HOST" in
  nano4|nano5) ;;
  *)
    echo "error: host must be nano4 or nano5"
    exit 2
    ;;
esac

case "$KIND" in
  ncu|nsys|csv|logs|raw|encoded|runs|misc) ;;
  *)
    echo "error: invalid kind: $KIND"
    exit 2
    ;;
esac

DATE="$(date +%F)"
SAFE_NAME="$(echo "$NAME" | tr ' /:' '___' | tr -cd 'A-Za-z0-9._+-')"
DEST="$HOME/artifacts/hpc/$HOST/$KIND/${DATE}__${SAFE_NAME}"

echo "[validate] $HOST:$REMOTE_PATH"
ssh "$HOST" "test -e '$REMOTE_PATH' && du -sh '$REMOTE_PATH'"

mkdir -p "$DEST"

echo "[pull] $HOST:$REMOTE_PATH"
echo "[dest] $DEST"

if ssh "$HOST" "test -d '$REMOTE_PATH'"; then
  rsync -avP --partial --append-verify \
    "$HOST:${REMOTE_PATH%/}/" \
    "$DEST/"
else
  rsync -avP --partial --append-verify \
    "$HOST:$REMOTE_PATH" \
    "$DEST/"
fi

{
  echo "source_host=$HOST"
  echo "source_path=$REMOTE_PATH"
  echo "dest_host=wsl"
  echo "dest_path=$DEST"
  echo "artifact_kind=$KIND"
  echo "name=$SAFE_NAME"
  echo "pulled_at=$(date -Is)"
  echo "transfer_tool=rsync"
} > "$DEST/transfer_manifest.txt"

find "$DEST" -maxdepth 3 -type f | sort > "$DEST/file_list.txt"
du -sh "$DEST" > "$DEST/size.txt"

echo
echo "[verify]"
cat "$DEST/size.txt"

case "$KIND" in
  ncu)
    echo
    echo "[ncu reports]"
    find "$DEST" -name "*.ncu-rep" | sort
    ;;
  csv)
    echo
    echo "[csv files]"
    find "$DEST" -name "*.csv" | sort
    ;;
  logs)
    echo
    echo "[log files]"
    find "$DEST" \( -name "*.out" -o -name "*.err" -o -name "*.log" -o -name "*.txt" \) | sort | head -50
    ;;
esac

echo
echo "[done] WSL artifact path:"
echo "$DEST"

echo
echo "[mac pull command]"
if [ "$KIND" = "ncu" ]; then
  echo "mkdir -p \"\$HOME/Downloads/ncu/$SAFE_NAME\""
  echo "rsync -avP \"wsl:$DEST/\" \"\$HOME/Downloads/ncu/$SAFE_NAME/\""
  echo "find \"\$HOME/Downloads/ncu/$SAFE_NAME\" -name \"*.ncu-rep\" | sort"
else
  echo "mkdir -p \"\$HOME/Downloads/hpc-artifacts/$SAFE_NAME\""
  echo "rsync -avP \"wsl:$DEST/\" \"\$HOME/Downloads/hpc-artifacts/$SAFE_NAME/\""
fi
```

Install:

```bash
mkdir -p ~/bin
chmod +x ~/bin/hpc-pull-artifact
```

Make sure `~/bin` is on `PATH`:

```bash
echo "$PATH" | tr ':' '\n' | grep -x "$HOME/bin" || echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc
```

---

## Full Operator Flow

Use this flow when the user says something like:

- "幫我把 nano4 的 NCU report 拉回來"
- "我要把 nano5 的大 CSV 拉到 WSL"
- "這次 job 的結果不要放 GitHub，放 WSL"
- "給我 Mac 可以拉回來看的指令"

### Flow

1. Ask:
   - Which host: `nano4` or `nano5`?
   - What remote path?

2. If the user does not know the path, provide the HPC-side agent prompt from Step 1a.

3. Validate the path from WSL.

4. Infer artifact kind.

5. Run `hpc-pull-artifact`.

6. Verify files landed correctly.

7. Return:
   - WSL artifact path
   - size
   - key files found
   - Mac pull command
   - whether Git was untouched

---

## Report Template

After transfer, report:

```text
Transfer complete.

Source:
- Host: <nano4|nano5>
- Path: <remote_path>

WSL destination:
- <dest>
- Size: <du -sh>

Detected files:
- <important files>

Mac pull command:
<command>

Git status:
- No heavy artifacts were committed or staged.
```

If partial:

```text
Transfer partially completed.

What exists:
- <files found>

Problem:
- <missing expected files or failed validation>

Next action:
- Ask the HPC-side agent to verify <specific path/job/output>.
```

---

## Common Mistakes

### Using GitHub as artifact storage

Problem: Large profiler reports or datasets are committed to Git.

Fix: Move them to `~/artifacts/hpc/...`, add ignore rules, and only commit metadata or summaries.

### Pushing from HPC to WSL

Problem: `nano4`/`nano5` may not be able to connect back to WSL.

Fix: Always prefer WSL pulling from HPC via `rsync`.

### Pulling everything to Mac

Problem: Mac storage gets polluted with raw data and large intermediate outputs.

Fix: Only pull `.ncu-rep` or selected small analysis files to Mac. Keep raw/encoded/full runs on WSL or NAS.

### Losing the source path

Problem: Artifact exists on WSL but nobody remembers where it came from.

Fix: Always write `transfer_manifest.txt`.

### Guessing paths

Problem: Agent invents a path or pulls the wrong directory.

Fix: Always validate with `ssh "$HOST" "test -e '$REMOTE_PATH' && du -sh '$REMOTE_PATH'"`.

### Using `scp` by habit

Problem: Interrupted transfers restart from zero.

Fix: Use `rsync -avP --partial --append-verify`.

### Blind `git add .`

Problem: Heavy artifacts get staged.

Fix: Use explicit `git add` paths only.

---

## Red Flags

Never:

- Commit `.ncu-rep`, `.nsys-rep`, `.qdrep`, `.sqlite`, raw datasets, encoded datasets, or huge CSVs.
- Use `git add .` after artifact transfer.
- Delete the HPC source immediately after transfer unless the user explicitly says to.
- Delete the WSL copy after Mac pull.
- Assume `nano4` when user may mean `nano5`.
- Assume the Mac SSH alias is `wsl` if the user says it is `shwsl`.
- Report success without running verification commands.

Always:

- Ask for host if unknown.
- Ask for remote path if unknown.
- Use WSL pull, not HPC push.
- Use `rsync`.
- Store under `~/artifacts/hpc/<host>/<kind>/YYYY-MM-DD__<name>/`.
- Write `transfer_manifest.txt`.
- Provide a Mac pull command after successful transfer.
- Keep Git clean.
