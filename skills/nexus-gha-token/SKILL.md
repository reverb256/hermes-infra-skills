---
name: nexus-gha-token
description: Fix GitHub authentication failures on the nexus builder host when building nix flake inputs from private GitHub repos. Shows as 'Bad credentials' / 401 during 'nix build' on nexus. Use when 'just deploy' fails with GitHub 401 errors during hermes-agent or other flake input fetches.
disable-model-invocation: false
metadata:
  hermes:
    tags: [infrastructure, nixos, github, ci]
    related_skills: [gha-runner-unstick, nix-flake-hygiene]
---

# Nexus GitHub Token Fix

## Symptom

```
Build failed for zephyr. Last log lines:
{ "message": "Bad credentials", "status": "401" }
note: trace involved the following derivations:
  derivation 'hermes-agent-wrapped-0.19.0'
  derivation 'hermes-agent-0.19.0'
```

## Check current auth state

```bash
ssh nexus "cat ~/.config/gh/hosts.yml 2>/dev/null | grep oauth_token"
```

## Fix

1. On zephyr (where `gh` is authenticated), get a fresh token:
   ```bash
   gh auth token
   ```

2. Copy it to nexus:
   ```bash
   ssh nexus "mkdir -p ~/.config/gh && echo 'github.com:
       users:
           reverb256:
               oauth_token: <TOKEN>
       git_protocol: ssh
       oauth_token: <TOKEN>
       user: reverb256' > ~/.config/gh/hosts.yml"
   ```

3. Test:
   ```bash
   ssh nexus "gh auth status 2>&1"
   ```

## Alternative: Use a deploy token

Create a fine-grained PAT on GitHub with read-only access to `NousResearch/hermes-agent` and add it to nexus's git config:

```bash
ssh nexus "git config --global credential.helper store && echo 'https://user:<PAT>@github.com' > ~/.git-credentials"
```

## Pitfalls

- The token must have access to `NousResearch/hermes-agent` (a private repo)
- If the token is for a machine user, ensure the machine user is added as a collaborator
- `gh auth status` is the fastest way to verify — always do this first before generating new tokens
