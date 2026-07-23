---
name: vaultwarden-sops-fix
description: Diagnose and fix sops-nix decryption failures that block nixos-rebuild switch or cause services to fail with NAMESPACE/226. Handles stale secret registrations (like Z.AI or vaultwarden-admin-token). Use when `switch-to-configuration` hangs, vaultwarden shows status=226/NAMESPACE, or sops-install-secrets fails.
disable-model-invocation: false
metadata:
  hermes:
    tags: [infrastructure, nixos, secrets, sops]
    related_skills: [deployment-debugger, nixos-declarative-only]
---

# Vaultwarden / Sops Fix

## Symptoms

- `sudo journalctl -u vaultwarden | grep NAMESPACE` → `status=226/NAMESPACE`
- `sudo journalctl | grep sops-install-secrets` → `failed to decrypt`
- `switch-to-configuration switch` hangs for 120+ seconds and times out
- `ls /run/secrets/vaultwarden-admin-token` → No such file or directory

## Root Cause

A sops-nix secret file (`secrets/<category>/<name>.yaml`) exists in the repo but:
1. The age-encrypted file is corrupted or missing
2. The encryption key is unavailable on the target host
3. The service that needs the secret can't start without it → 226/NAMESPACE
4. The switch hangs waiting for the service to start → times out

## Procedure

### 1. Identify the failing secret

```bash
sudo journalctl -u vaultwarden --no-pager | grep "No such file\|NAMESPACE"
sudo journalctl | grep "sops-install-secrets" | grep "failed"
```

This tells you which `/run/secrets/<name>` is missing.

### 2. Find the sops registration

```bash
grep -n "<name>" /etc/nixos/modules/system/sops-secrets-registry.nix
```

This shows the `sopsFile` path and `path` (where it should be decrypted to).

### 3. Check if the source secret file exists

```bash
ls -la <sopsFile>   # e.g. secrets/selfhosting/vaultwarden-admin-token.yaml
```

If the file is missing or corrupted, the secret can't be decrypted.

### 4. Remove the registration (if the service is dead or the key is irrecoverable)

```nix
# In sops-secrets-registry.nix, remove the block like:
# "<name>" = {
#   sopsFile = "...";
#   path = "/run/secrets/<name>";
#   ...
# };
```

Also remove from `agenix-secrets-registry.nix` if present:

```nix
# <name> = {
#   file = "...";
#   ...
# };
```

### 5. Delete the orphaned secret file (optional)

```bash
sudo rm -f secrets/<category>/<name>.yaml
```

### 6. Rebuild and switch

```bash
cd /etc/nixos
git add -A && git commit -m "fix: remove stale <name> sops registration"
git push origin main
just deploy zephyr
```

## Verification

```bash
ls /run/secrets/<name>       # Should NOT exist anymore (no longer expected)
systemctl is-active vaultwarden  # May still fail if other issues exist
```

## Pitfalls

- **Never disable vaultwarden permanently** — it's a critical service. Only remove its sops registration if the underlying key is truly lost.
- **Check both registries** — sops-secrets-registry.nix AND agenix-secrets-registry.nix may both reference the same secret.
- **Check for other failing services** — vaultwarden is the most common blocker, but any service with a broken sops secret can block the switch.
