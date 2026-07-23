---
name: zram-sizing
description: Audit and recommend zram/zswap swap configuration for a host based on its workload, RAM size, and disk type. Covers zram vs zswap decision, memoryPercent sizing, swappiness tuning, and page-cluster. Use when diagnosing swap exhaustion, OOMs, or setting up a new host.
disable-model-invocation: false
metadata:
  hermes:
    tags: [infrastructure, performance, memory, swap]
    related_skills: [oom-defense, nixos-cluster-ops]
---

# ZRAM Sizing

## Current state

```bash
zramctl
cat /sys/module/zswap/parameters/enabled
cat /proc/sys/vm/swappiness
cat /proc/sys/vm/page-cluster
free -h
```

## Decision: zram vs zswap

| Factor | zram-only | zswap + disk swap |
|---|---|---|
| Disk wear | None | Low (NVMe) |
| Graceful degradation | No (hard cap) | Yes (tiers to disk) |
| Best for | Desktop, laptops | Servers, sustained load |
| Swappiness | 150-200 | 10-40 |

For this homelab's desktops: **zram-only** is recommended (no disk wear, simpler).

## ZRAM sizing formula

```
memoryPercent = (max_expected_swap_usage / total_ram) * 100
```

For build workloads:
- 31GB host with occasional 7-8GB builds → 40% (~12GB)
- 15GB host (forge) with mining + builds → 50% (~7.5GB)
- General desktop with browsers → 25-40%

## Tuning

| Setting | zram value | zswap value |
|---|---|---|
| vm.swappiness | 180 | 40 |
| vm.page-cluster | 0 | 3 |
| vm.vfs_cache_pressure | 50 | 150 |

## Verification

After changes:
- `zramctl` shows DISKSIZE matching `memoryPercent`
- `cat /proc/sys/vm/swappiness` matches desired value
- `free -h` shows swap with free space after a load cycle
- No OOM events in `journalctl -k | grep oom`
