---
name: secretspec-checkpoint
description: Audit migration progress from sops-nix/agenix to secretspec. Compares the sops-secrets-registry.nix and agenix-secrets-registry.nix against secretspec.toml declarations, reports gaps, and recommends next-phase actions.
disable-model-invocation: false
metadata:
  hermes:
    tags: [infrastructure, secrets, migration, audit]
    related_skills: [vaultwarden-sops-fix, nixos-declarative-only]
---

# Secretspec Migration Checkpoint

## Compare registries

```bash
cd /etc/nixos

echo "=== sops secrets ==="
grep "sopsFile" modules/system/sops-secrets-registry.nix | wc -l

echo "=== agenix secrets ==="
grep "ageSecret\|\.age" modules/system/agenix-secrets-registry.nix | grep -v "^\s*#" | wc -l

echo "=== secretspec declared ==="
grep -c "^[A-Z]" ~/Projects/secretspec/secretspec.toml 2>/dev/null || echo "(secretspec.toml not in cluster repo)"
```

## Check each secret's status

For each secret in the registries, determine:

| Status | Meaning | Count |
|---|---|---|
| ✅ Declared in secretspec.toml | Listed in the Phase 1 declaration | X |
| ❌ Missing from secretspec | Needs adding | X |
| 🔴 Broken sops file | Exists in registry but file can't be decrypted | X |
| 🟢 Age file exists but not moved | Ready for Phase 2 migration | X |

## Find broken secrets

```bash
# Check for sops decryption failures
sudo journalctl | grep "sops-install-secrets" | grep "failed"

# Check for missing secret files
for f in $(grep "sopsFile" modules/system/sops-secrets-registry.nix | grep -oP '"\K[^"]+\.yaml'); do
  [ ! -f "$f" ] && echo "MISSING: $f"
done
```

## Recommended actions per finding

- **Broken secret** → Remove from both registries, delete `.yaml` file (see `vaultwarden-sops-fix`)
- **Missing from secretspec** → Add to `~/Projects/secretspec/secretspec.toml`
- **Ready for Phase 2** → No action until PR #58 or the in-house crate ships

## Verify secretspec is installed

```bash
~/.local/bin/secretspec --version
secretspec check --profile development
```

If `secretspec check` fails, the `.env.secrets` file may need populating.
