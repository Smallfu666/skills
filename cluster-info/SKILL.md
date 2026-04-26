---
name: cluster-info
description: Provides TWCC/NCHC cluster specs, partitions, QoS limits, GPU types, MinGPU, SU billing, pricing, and architecture details. Use when needing sinfo, scontrol, partition, QoS, MinGPU, GPU type, SU billing, NTD cost, TWCC pricing, ARM/x86, GB200, H100/H200, nano4, nano5, Taiwania III, or cluster identification.
---

# TWCC / NCHC Cluster Info

This skill provides cluster-specific hardware, partition, QoS, billing, and architecture data.

## Quick start

Use the cached cluster info only when it matches the current cluster. If the cluster is unknown or the cache is stale, refresh it from Slurm and sacctmgr first.

## Workflows

1. Confirm which cluster is in use before giving partition, QoS, GPU, billing, or job-submission advice.
2. Use live Slurm and sacctmgr queries to refresh cluster tables when the cache is missing or stale.
3. Fetch official NCHC pricing before quoting NTD cost.
4. Treat ARM/GB200/Grace-Hopper nodes as architecture-sensitive and warn about binary compatibility.

## Advanced details

See [REFERENCE.md](REFERENCE.md) for the full rules, commands, cache format, and decision flow.
