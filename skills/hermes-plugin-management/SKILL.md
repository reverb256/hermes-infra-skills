---
name: hermes-plugin-management
description: "Manage versioned Hermes plugins, skills, profiles."
version: 0.1.0
author: Hermes Agent
license: MIT
platforms: [linux]
metadata:
  hermes:
    tags: [Plugins, Skills, Profiles, Versioning, Pack]
---

# Hermes Plugin Management

Keeps every Hermes customization versioned and reproducible: plugins in
`reverb256/hermes-plugins`, skills in `hermes-infra-skills` / `hermes-skills`,
profiles in `hermes-profiles`. The critical gap this solves: plugins are
local-only by default and `hermes plugins pack export` cannot pin them until
they have git provenance (installed from a repo, not a manual directory copy).

## When to Use

- "Make sure our Hermes plugins/extensions are properly managed"
- "Add a new plugin and make it reproducible"
- "Restore Hermes plugins on a fresh machine"
- "What's versioned vs local-only in ~/.hermes?"
- "Pin our plugin set to exact commit SHAs"

## Prerequisites

- GitHub CLI authenticated (`gh`) for repo creation/push.
- The `hermes` CLI on PATH.
- gitlawb (`gl`) optional — repos have both `gitlawb://` and GitHub `origin` remotes; GitHub is the working push path (gitlawb daemon may be down).

## How to Run

Version a new plugin:

```bash
# 1. Add it to the repo
cd ~/Projects/hermes-plugins
mkdir -p plugins/<name>
cp ~/.hermes/plugins/<name>/{plugin.yaml,__init__.py} plugins/<name>/
git add -A && git commit -m "feat: add <name> plugin" && git push origin main

# 2. Give the live copy git provenance (reinstall from repo)
hermes plugins install reverb256/hermes-plugins/plugins/<name> --force

# 3. Re-pin the pack
hermes plugins pack export | grep -v '^# WARNING' > pack.yaml
git add pack.yaml && git commit -m "chore: re-pin pack" && git push origin main
```

Restore everything on a fresh machine:

```bash
git clone git@github.com:reverb256/hermes-plugins.git
cd hermes-plugins
hermes plugins pack install pack.yaml
```

## Quick Reference

- Plugin repo: `github.com/reverb256/hermes-plugins` (private) — `plugins/<name>/{plugin.yaml,__init__.py}`
- Pack: `pack.yaml` at repo root — pins each plugin to `repo + ref (SHA) + subdir`
- Install one: `hermes plugins install reverb256/hermes-plugins/plugins/<name> [--force] [--no-enable]`
- Export pack: `hermes plugins pack export`
- Restore: `hermes plugins pack install pack.yaml`
- Skills: `hermes skills tap add reverb256/hermes-infra-skills` then `hermes skills install <repo>/<skill>`
- Profiles: `reverb256/hermes-profiles` — `distribution.yaml` + `profiles/<name>/`

## Procedure

1. **Inventory** — `ls ~/.hermes/plugins/` and check provenance: `hermes plugins pack export` lists any plugin with "no Git provenance" as a comment (not installable). Those are the unmanaged ones.
2. **Add to repo** — copy plugin files into `~/Projects/hermes-plugins/plugins/<name>/`, commit, push to GitHub.
3. **Reinstall from repo** — `hermes plugins install reverb256/hermes-plugins/plugins/<name> --force` (this records git provenance). `--no-enable` then re-`hermes plugins enable <name>` if you want to control activation.
4. **Re-pin pack** — `hermes plugins pack export`, strip `# WARNING` comment lines, save as `pack.yaml`, commit+push.
5. **Verify** — `hermes plugins pack show pack.yaml` (dry-run parse) lists every plugin with its pinned ref.
6. **On a fresh machine** — clone + `hermes plugins pack install pack.yaml`; each plugin's capabilities still require per-plugin consent.

## Pitfalls

- **Security scan false positives.** `hermes plugins install` runs a security scan; reading `/proc/self/fd` (tty resolution) or `subprocess.run` flags CAUTION and BLOCKS. For your own plugins these are legitimate; use `--force` to override.
- **`--force` wipes + reinstalls.** It removes the existing plugin dir first. After reinstall the plugin is DISABLED unless you pass `--enable` or re-enable.
- **Subdir syntax matters.** The installer supports `owner/repo/subdir`; without the subdir it looks for a plugin at the repo root (`community`) and fails to find yours.
- **Stale warnings in pack export.** The `# WARNING (not exported)` header lines can list plugins that ARE actually pinned below — re-run export after reinstall; trust the `plugins:` section, not the header comments.
- **`memory` and `mnemosyne` are special** — memory has no plugin.yaml (empty stub), mnemosyne is a venv symlink. Neither belongs in the repo.
- **memlawb is its own repo** (`reverb256/memlawb-for-hermes`) — do not move it into hermes-plugins.
- **Plugin handler `**kwargs`.** Any plugin registering a tool must define `handler(args, **kwargs)` — Hermes calls `handler(args, **kwargs)` passing `task_id`. A plain `def handler(args)` errors with "task_id parameter drifted".

## Verification

After any change, `hermes plugins pack show pack.yaml` must list every plugin with a pinned ref, and `hermes plugins list` must show each as `enabled`. On a fresh checkout, `hermes plugins pack install pack.yaml` restores the identical set.

## Reference

The full surface map (every Hermes customization, its versioned home, restore path, and the remaining gaps) lives in `hermes-infra-skills/HERMES-MANAGEMENT-MAP.md` — load it before migrating or restoring a host.
