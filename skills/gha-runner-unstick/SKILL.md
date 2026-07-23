---
name: gha-runner-unstick
description: Diagnose and restart a stalled self-hosted GitHub Actions runner on NixOS/K3s. Handles token expiry, registration issues, and the 'startup_failure' CI conclusion. Use when GitHub Actions shows 'startup_failure', the runner service is inactive, or jobs queue without running.
disable-model-invocation: false
metadata:
  hermes:
    tags: [infrastructure, ci, github-actions]
    related_skills: [deployment-debugger, nixos-cluster-ops]
---

# GitHub Actions Runner Unstick

## Check runner state

```bash
ssh nexus "systemctl is-active github-actions-runner"
ssh nexus "systemctl status github-actions-runner --no-pager | head -10"
```

If `inactive (dead)`, proceed.

## Check the token

The runner token is stored in a sops-nix secret. Verify it exists:

```bash
ssh nexus "ls -la /run/secrets/github-runner-pat"
```

If missing, the sops file needs fixing (see `vaultwarden-sops-fix` for the pattern).

## Restart the runner

```bash
ssh nexus "sudo systemctl restart github-actions-runner"
sleep 5
ssh nexus "systemctl is-active github-actions-runner"
```

## Verify

- `gh run list --limit 3` on the affected repo should show queued jobs transitioning to `in_progress`
- The runner status page at `https://github.com/reverb256/nixos-config/settings/actions` should show the runner as "Idle"

## Pitfalls

- The runner requires network access to `github.com`. Check DNS/firewall.
- Token rotation: GitHub tokens expire after 1 year. If the runner fails immediately after registration, the token may be expired.
- Fish shell quoting: if the runner uses fish for shell, ensure commands in the runner's `.env` use `bash --norc --noprofile -c` wrappers.
