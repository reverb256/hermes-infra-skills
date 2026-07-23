# Hermes Infra Skills

Infrastructure operations skills for Hermes Agent on the reverb256 homelab.

## Skills

| Skill | Description |
|---|---|
| `oom-defense` | Audit 3-layer OOM protection (systemd-oomd, earlyoom, oom_score_adj), zswap/zram architecture, swappiness tuning |
| `drift-cleanup` | Find and remediate hand-placed /etc files outside the NixOS declarative tree |
| `vaultwarden-sops-fix` | Diagnose sops-nix decryption failures that block nixos-rebuild switch |
| `gha-runner-unstick` | Restart stalled self-hosted GitHub Actions runner, diagnose token issues |
| `nexus-gha-token` | Fix GitHub 401 on nexus builder during flake input fetches |
| `zram-sizing` | Audit and recommend zram vs zswap configuration for host workload |
| `secretspec-checkpoint` | Compare sops/agenix registry vs secretspec.toml, report migration gaps |
| `daily-oom-audit` | Blueprint: daily cluster-wide OOM health check |
| `weekly-drift-scan` | Blueprint: weekly config drift scan across all hosts |

## Install

```bash
hermes skills tap add reverb256/hermes-infra-skills
hermes skills install reverb256/hermes-infra-skills/oom-defense
```
