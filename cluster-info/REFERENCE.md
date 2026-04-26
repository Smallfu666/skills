# TWCC / NCHC Cluster Info

The user may work on multiple NCHC environments, especially:

- `nano4`
- `nano5`
- `taiwania3` / `t3-c4`

Never assume these have the same partitions, GPUs, QoS, wall-time limits, or billing behavior.

## 1. Cluster Identification — Never Guess

Before giving partition, QoS, GPU, billing, or job-submission advice, confirm the current cluster.

Valid cluster identifiers include:

```text
nano4
nano5
taiwania3
t3-c4
twcc
```

Do not infer cluster identity from:

```bash
hostname
scontrol show config | grep ClusterName
```

Reason:

- `hostname` returns login-node names, not cluster names.
- `ClusterName` may return generic values like `hpc`.

If memory does not already contain a confirmed current cluster, ask:

```text
Which cluster are you on? nano4, nano5, or taiwania3/t3-c4?
```

Do not continue to partition/QoS advice until the cluster is confirmed.

## 2. Use Per-Cluster Cached Info When Valid

Partition and QoS layouts are stable enough to cache, but the cache must be cluster-specific.

Check memory for an entry named:

```text
TWCC/NCHC cluster info
```

Use cached data only if:

- the cached cluster matches the current cluster;
- the entry has a query date;
- the user has not reported conflicting partition/GPU/QoS behavior;
- the user has not explicitly requested a refresh.

Refresh the cache if:

- no cache exists;
- cache is for another cluster;
- cluster changed, e.g. nano4 → nano5;
- user reports a mismatch;
- query date is missing;
- user requests fresh data.

## 3. Live Query Commands for Refresh

When cache is missing or stale, run all four commands.

```bash
# Hardware layout
sinfo -o "%20P %8c %8m %30G %10l %6D %20f" --noheader

# Partition details
scontrol show partition

# QoS limits
sacctmgr show qos format=Name,Priority,MinTRESPerJob,MaxTRESPerUser,MaxTRESPerJob,MaxWall,MaxJobsPerUser -P

# User account / association limits
sacctmgr show assoc where user=$USER \
  format=Account,Cluster,Partition,GrpTRES,MaxTRES,MaxJobs,GrpTRESRunMins -P
```

Important:

- `sinfo` gives partition hardware, wall time, node count, features.
- `scontrol show partition` gives partition → QoS mapping.
- `sacctmgr show qos` is the authoritative source for `MinTRESPerJob`.
- `MinGPU` must be derived from:

```text
partition → QoS → MinTRESPerJob
```

Example:

```text
normal → p_normal → MinTRESPerJob=gres/gpu=64 → MinGPU=64
```

If `MinTRESPerJob` is empty, no minimum GPU requirement is configured.

## 4. Required Cached Table Format

After live query, immediately save one consolidated memory entry.

Replace older entries for the same cluster.

```markdown
TWCC/NCHC cluster info
queried: YYYY-MM-DD
cluster: <nano4 | nano5 | taiwania3 | t3-c4>

| Partition | GPU | GPUs/node | MinGPU/job | MaxGPU/job | MaxGPU/user | CPUs/node | Mem/node | Max wall | MaxJobs/user | Nodes | Arch | Notes |
|---|---:|---:|---:|---:|---:|---:|---:|---|---:|---:|---|---|
| dev | ... | ... | — | — | ... | ... | ... | ... | ... | ... | x86_64/aarch64 | ... |
| normal | ... | ... | ... | ... | ... | ... | ... | ... | ... | ... | x86_64/aarch64 | MinGPU from QoS |

Node ranges:
- <GPU type>: <hostname-prefix>[range]

User accounts:
- <account info from sacctmgr assoc>

Known bad nodes:

| Node | Issue | Date Added | Expiry |
|---|---|---|---|
| ... | ... | ... | ... |
```

Conventions:

```text
— = no configured limit or not set
```

Known bad nodes older than 7 days must be re-verified before excluding them.

## 5. Live Status Checks — Never Cache

Always query these fresh.

```bash
# Remaining SU budget / run-minute limits
sacctmgr show assoc where user=$USER format=Account,GrpTRESRunMins -P

# Current user queue
squeue -u $USER -o "%.18i %.9P %.30j %.8u %.8T %.10M %.10l %.6D %R"
```

Do not store queue state or remaining budget as persistent memory.

## 6. Pricing — Always Fetch Official Source

Never quote hardcoded prices.

Before giving NTD cost estimates, fetch current pricing from official NCHC sources:

Primary:

```text
https://iservice.nchc.org.tw/nchc_service/nchc_service_qa.php?target=54
```

Backup:

```text
https://www.twcc.ai/doc?page=price
```

If official pricing is unavailable, ask the user to confirm:

- GPU type;
- project category;
- current GPU-hour rate.

Billing formula:

```text
GPU-hours = execution_hours × requested_GPUs
cost_NTD = GPU-hours × rate_per_GPU_hour
```

## 7. Architecture Rules

Detect architecture from `sinfo` features.

```text
aarch64 → ARM
otherwise usually x86_64
```

ARM / GB200 / Grace-Hopper:

- x86 binaries will not run.
- Many conda packages, PyPI wheels, and compiled CLI tools may fail.
- Test compatibility with a short dev job first.
- Prefer ARM-native containers.

x86 / H100 / H200:

- Standard Linux, conda, PyPI, and CLI workflows are usually safe.
- Prefer these nodes unless explicitly benchmarking ARM/GB200 behavior.

## 8. Default Decision Flow

```text
1. Confirm cluster name.
2. Check per-cluster cached table.
3. If cache is valid, use it.
4. If cache is missing/stale, run sinfo + scontrol + sacctmgr QoS + sacctmgr assoc.
5. Save consolidated cluster info to memory.
6. For current queue or SU budget, always query live.
7. For pricing, always fetch official source before quoting NTD cost.
8. For ARM nodes, warn about binary compatibility before suggesting tools or jobs.
```
