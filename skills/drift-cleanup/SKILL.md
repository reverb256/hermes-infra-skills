---
name: drift-cleanup
description: Find and remediate hand-placed files in /etc that bypass the NixOS declarative source tree — sysctl.d files, modprobe.d files, systemd drop-ins, and other admin-created config that will be lost on rebuild. Use when auditing config drift or after finding an /etc file that doesn't come from the store.
disable-model-invocation: false
metadata:
  hermes:
    tags: [infrastructure, nixos, audit, hygiene]
    related_skills: [nixos-declarative-only, deployment-debugger, nixos-cluster-ops]
---

# Drift Cleanup

Find and remove hand-placed files outside the NixOS declarative tree.

## Scan for drift

```bash
# Check sysctl.d for non-store files
for f in /etc/sysctl.d/*.conf; do
  case $(readlink -f "$f") in /nix/store*) ;; *) echo "DRIFT: $f";; esac
done

# Check modprobe.d for non-store files
for f in /etc/modprobe.d/*.conf; do
  case $(readlink -f "$f") in /nix/store*) ;; *) echo "DRIFT: $f";; esac
done

# Check systemd unit/drop-in dirs
for d in /etc/systemd/system; do
  find "$d" -maxdepth 1 ! -type l ! -name "*.wants" 2>/dev/null
done
```

## Categorize each drift file

| Category | Action | Example |
|---|---|---|
| **Needed, not in NixOS** | Migrate to NixOS module | `nvidia.conf` → `boot.extraModprobeConfig` |
| **Redundant** | Delete | `99-slab-cache.conf` (overridden by NixOS) |
| **Dead config** | Delete | Old Cilium rp_filter override on flannel cluster |
| **Hotfix artifact** | Delete | `/etc/earlyoom/earlyoom.conf` (NixOS unit ignores it) |

## Procedure

1. Run the scan above
2. For each DRIFT file, determine its category
3. If needed: add the equivalent setting to a NixOS module and deploy
4. Delete the hand-placed file
5. Re-apply system services: `sudo systemctl restart systemd-sysctl`

## Pitfalls

- Don't delete files that are load-bearing (like Cilium rp_filter on a Cilium cluster)
- Some `/etc/systemd/system` symlinks to `/dev/null` are NixOS-generated masks — NOT drift
- Check `ls -la /etc/static/` — files there are NixOS-managed, never drift
