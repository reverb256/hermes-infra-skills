# Hermes Customization Management Map

> **Snapshot:** 2026-08-21 · This is the durable index of every Hermes
> customization surface, where it is versioned, and how to restore it.
> Preserve this as the omarchy/Reverb-OS migration proceeds — it is the
> Layer-2/3 user-side state map.

## The one-line story

Everything Hermes-related is versioned in GitHub repos under `reverb256`,
except three surfaces still being closed: **MCP bridge scripts (24 files),
remaining `.env` secrets (5 keys), and cron jobs (1)**.

## Surface map

| Surface | Live location | Versioned home | Restore | Status |
|---|---|---|---|---|
| **Plugins (6)** | `~/.hermes/plugins/` | `reverb256/hermes-plugins` + `pack.yaml` | `hermes plugins pack install pack.yaml` | ✅ done |
| **Skills (infra/agent)** | `~/.hermes/skills/` | `reverb256/hermes-infra-skills` | `hermes skills tap add` + `install <repo>/<skill>` | ✅ done |
| **Skills (general)** | `~/.hermes/skills/` | `reverb256/hermes-skills` | same | ✅ done |
| **Profiles** | `~/.hermes/profiles/` | `reverb256/hermes-profiles` (`distribution.yaml`) | clone + enable | ✅ done |
| **Secrets (5 wired)** | `~/.hermes/.env` | private flake `reverb256/nixos-secrets` → `hermes-config-secrets.service` renders at boot | `just deploy zephyr` (pending) | 🟡 needs deploy |
| **Secrets (5 unwired)** | `~/.hermes/.env` | ❌ no sops file | — | ❌ gap |
| **Cron (1)** | `hermes cron list` | ❌ no export | — | ❌ gap |
| **MCP bridges (24)** | `/data/agents/mcp-bridges/` | ❌ not in git (but Nix-declared) | — | ❌ gap |
| **Gateway/A2A** | `~/.hermes/config.yaml` | Nix-rendered (`hermes-config-emit.service`) | deploy | ✅ |
| **memlawb plugin** | `~/.hermes/plugins/memlawb` | `reverb256/memlawb-for-hermes` (own repo) | git clone | ✅ |
| **memory / mnemosyne** | `~/.hermes/plugins/` | special cases (stub / venv symlink) | n/a | ⚠️ |

## Repos (all private GitHub + gitlawb remote)

- `reverb256/hermes-plugins` — plugins + `pack.yaml`
- `reverb256/hermes-infra-skills` — infra/agent skills incl. `hermes-plugin-management`, `hermes-live-session-intercom`
- `reverb256/hermes-skills` — general skill collection
- `reverb256/hermes-profiles` — profile distribution
- `reverb256/memlawb-for-hermes` — memlawb plugin + provider
- `reverb256/nixos-secrets` — private secrets flake (sops YAMLs)

Push path: use the GitHub `origin` (`git@github.com:reverb256/<repo>.git`).
The `gitlawb://` remote exists but the gitlawb daemon may be down.

## Secrets detail

`~/.hermes/.env` is rendered by `hermes-config-secrets.service`
(`modules/services/hermes-secrets.nix`) from sops files at boot, via
`secretspecEnvVarMappings` in `hosts/zephyr/configuration.nix` (~line 1373).

**Wired (5):** `NVIDIA_API_KEY`, `OPENCODE_ZEN_API_KEY`,
`OPENCODE_GO_API_KEY`, `EXA_API_KEY`, `GITHUB_TOKEN` — all have sops files in
`nixos-secrets/secrets/` + secretspec routes.

**Unwired (hand-placed in `.env` only, no sops file):** `KILOCODE_API_KEY`,
`COMMANDCODE_API_KEY`, `GOOGLE_API_KEY`, `MNEMOSYNE_*`, `TERMINAL_SSH_*`.

**Deploy gotcha:** the running zephyr generation (2026-08-15 era,
`al3lcpvk...`) predates the secretspec migration — `/run/secrets/` is EMPTY,
so even the 3 original keys aren't provisioned. `just deploy zephyr` activates
a generation that provisions them. This is the known old-generation-lag pattern
(documented in the garage skill).

## MCP bridges

24 scripts in `/data/agents/mcp-bridges/` (agentmemory, browser-use, casdoor,
cloudflare, context7, cua-driver, exa, github, gitlab?, etc.). NOT in git, but
referenced declaratively in nixos-config:
- `modules/development/ai-coding-tools/mcp-defs.nix`
- `modules/development/mcp-local-servers.nix`
- `modules/services/mcp-server-registry.nix`

**Action:** move the bridge scripts into a git repo (fold into nixos-config or
a dedicated `mcp-bridges` repo) so they're declaratively installable, mirroring
the plugin pattern.

## Cron

1 job: `site-agency-pipeline` (`0 9 * * *`, workdir `~/Projects/site-agency`).
No export command exists. **Action:** write a `cron.yaml` declarative spec +
restore procedure.

## Fresh-machine bootstrap (end state)

One command that restores: plugins (pack install) + skills (tap+install) +
profiles (distribution) + secrets (deploy) + cron (cron.yaml) + MCP (bridges
repo). This is the Layer-2/3 equivalent of the Omarchy migration story — a new
host becomes "your Hermes" instantly.

## Gotchas (from the build, 2026-08-21)

- Plugin tool handlers MUST accept `**kwargs` (`handler(args, **kwargs)`); a
  plain `def handler(args)` errors "task_id parameter drifted".
- `hermes plugins install` security scan flags `/proc/self/fd` reads and
  `subprocess.run` as CAUTION → needs `--force` for your own plugins.
- `hermes plugins install owner/repo/plugins/<name>` — the subdir syntax is
  required; without it the installer looks for `community` at repo root.
- `--force` wipes + disables the plugin; re-enable after.
- `hermes plugins pack export` header `# WARNING (not exported)` lines can be
  stale — trust the `plugins:` section, not the header comments.
