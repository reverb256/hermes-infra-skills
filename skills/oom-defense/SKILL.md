---
name: oom-defense
description: Audit and remediate OOM (out-of-memory) protection across the NixOS/K3s homelab. Covers the 3-layer defense (systemd-oomd, earlyoom, oom_score_adj), zswap/zram swap architecture, and swappiness tuning. Use when a host OOM-kills processes, swap is exhausted, or the desktop session is killed under memory pressure.
disable-model-invocation: false
metadata:
  hermes:
    tags: [infrastructure, nixos, performance, oom]
    related_skills: [nixos-cluster-ops, deployment-debugger, gpu-mining-operations]
---

# OOM Defense Audit

Diagnose and fix the 3-layer OOM protection: earlyoom, systemd-oomd, and oom_score_adj.

## Layer 1 — earlyoom (primary, fastest)

earlyoom uses percentage-based memory + swap thresholds. It reacts faster than systemd-oomd's PSI polling for browser-heavy desktops.

### Check

```bash
systemctl is-active earlyoom
pgrep -af "bin/earlyoom"
```

Args should include `--avoid` for the desktop session and `--prefer` for reloadable processes (browser content, nix builds):

```
-m 12 -s 50 --prefer '(Web Content|Isolated Web|nix)' --avoid '(niri|noctalia|zen|spotify|vesktop|opencode|hermes|Xwayland|pipewire)'
```

### Fix if missing

Add to the host's `services.earlyoom.extraArgs` in NixOS config:

```nix
services.earlyoom = {
  extraArgs = [
    "--prefer" "(Web Content|Isolated Web|nix)"
    "--avoid"  "(niri|noctalia|zen|spotify|vesktop|opencode|hermes|Xwayland|pipewire)"
  ];
};
```

## Layer 2 — systemd-oomd (passive backstop)

systemd-oomd uses cgroup-level PSI monitoring. It does NOTHING unless a slice has `ManagedOOMSwap=kill` set.

### Check

```bash
systemctl is-active systemd-oomd
systemctl show -.slice --property=ManagedOOMSwap
cat /run/current-system/etc/systemd/oomd.conf
```

Expected:
- `systemd-oomd` active
- `-.slice` has `ManagedOOMSwap=kill`
- `oomd.conf` has `[OOM] SwapUsedLimit=90,
MemoryUsedLimit=90`

### Fix if missing

Enable root slice opt-in:

```nix
systemd.oomd = {
  enable = true;
  settings.OOM = {
    SwapUsedLimit = 90;
    MemoryUsedLimit = 90;
  };
};
systemd.slices."-".sliceConfig = {
  ManagedOOMSwap = "kill";
};
```

## Layer 3 — oom_score_adj (process-level protection)

Processes important to the desktop session should have `oom_score_adj` set to a negative value, making them less likely to be killed.

### Check

```bash
for p in niri zen spotify vesktop opencode; do
  pid=$(pgrep -f "$p" | head -1)
  [ -n "$pid" ] && echo "$p: $(cat /proc/$pid/oom_score_adj)"
done
```

Expected: `-500` for protected processes. If `> 0`, the process is MORE likely to be killed — wrong.

### Fix

Relies on earlyoom's `--avoid` flag (Layer 1), which subtracts 300 from the oom_score at decision time. If earlyoom is correctly configured with `--avoid` for the desktop session, this layer is covered.

## Swap Architecture — zswap vs zram

The most common OOM trigger on this cluster is zswap+zram running simultaneously — a documented anti-pattern. zswap intercepts pages before they reach zram, they fight over capacity, and when zram's hard cap fills there's no disk backstop → swap exhaustion → OOM.

### Check

```bash
cat /sys/module/zswap/parameters/enabled   # Y = BAD when zram is active
zramctl                                    # check DISKSIZE vs memoryPercent
cat /proc/sys/vm/swappiness                # >100 for zram-only, ~40 for zswap
```

### Fix

Pick one: **zram-only** (desktop, no disk wear) or **zswap + disk swap** (server, graceful degradation).

For zram-only (recommended for this cluster's desktops):

```nix
boot.kernelParams = [ "zswap.enabled=0" ];
zramSwap = {
  enable = true;
  algorithm = "zstd";
  memoryPercent = 40;  # ~12GB on 32GB host
  priority = 999;
};
boot.kernel.sysctl = {
  "vm.swappiness" = 180;
  "vm.page-cluster" = 0;
};
```

For zswap + disk swap:

```nix
zramSwap.enable = false;
# Add kernel params for zswap (kernel-hardening module enables it)
swapDevices = [{ device = "/swapfile"; size = 16384; }];
boot.kernel.sysctl = {
  "vm.swappiness" = 40;
};
```

## Verification

After changes:
1. `systemctl is-active earlyoom` → active
2. `systemctl is-active systemd-oomd` → active
3. `pgrep -af earlyoom | grep -o 'avoid.*pipewire'` → non-empty
4. `cat /proc/sys/vm/swappiness` → 180 (zram) or 40 (zswap)
5. `cat /sys/module/zswap/parameters/enabled` → N
6. `free -h` → swap should have free space after a build cycle
