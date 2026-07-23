---
name: weekly-drift-scan
description: Weekly scan for config drift across the cluster — hand-placed /etc files, stale sysctl overrides, unmanaged systemd drop-ins, and NixOS config hygiene. Produces remediation report.
disable-model-invocation: true
metadata:
  hermes:
    tags: [infrastructure, audit, hygiene, cron]
    related_skills: [drift-cleanup, nixos-declarative-only, nixos-cluster-ops]
    blueprint:
      schedule: "0 10 * * 1"
      deliver: origin
      prompt: "Run a config drift scan across the cluster. Check for hand-placed files in /etc/sysctl.d, /etc/modprobe.d, /etc/systemd/system on zephyr, nexus, forge, and sentry. Compare against the NixOS source tree. Report any drift files found and whether they have a corresponding NixOS declaration."
      no_agent: false
---

# Weekly Drift Scan (Blueprint)

This skill is a scheduled blueprint. It scans for config drift across all cluster hosts.

## Scan steps

1. Check sysctl.d for hand-placed files:
   ```bash
   for h in zephyr nexus forge sentry; do
     echo "=== $h ==="
     ssh $h "for f in /etc/sysctl.d/*.conf; do case \$(readlink -f "\$f") in /nix/store/*) ;; *) echo "DRIFT: \$f";; esac; done"
   done
   ```

2. Check modprobe.d:
   ```bash
   for h in zephyr nexus forge sentry; do
     echo "=== $h ==="
     ssh $h "for f in /etc/modprobe.d/*.conf; do case \$(readlink -f "\$f") in /nix/store/*) ;; *) echo "DRIFT: \$f";; esac; done"
   done
   ```

3. Check systemd unit files not in store:
   ```bash
   for h in zephyr nexus forge sentry; do
     echo "=== $h ==="
     # Check for non-store unit files (skip /dev/null masks which are nixos-generated)
     ssh $h "find /etc/systemd/system -maxdepth 1 -type f 2>/dev/null | while read f; do case \$(readlink -f "\$f") in /nix/store/*|/dev/null) ;; *) echo "DRIFT: \$f";; esac; done"
   done
   ```

## Output

Produce a report with:
- Per-host drift file inventory
- Severity: HIGH if the drift overrides a NixOS-managed value, LOW if it's independent
- Recommended action: migrate to NixOS, delete if stale, or keep if load-bearing
- Cross-reference most recent git diff in /etc/nixos to see if the drift is already handled

If no drift found, report that and close.
