---
name: daily-oom-audit
description: Daily audit of OOM protection health across all cluster hosts — check earlyoom status, swap usage, zram health, and recent OOM kills. Produces a summary table.
disable-model-invocation: true
metadata:
  hermes:
    tags: [infrastructure, monitoring, oom, cron]
    related_skills: [oom-defense, miner-audit, deployment-debugger]
    blueprint:
      schedule: "0 9 * * *"
      deliver: origin
      prompt: "Run a full OOM protection audit across the cluster. Check earlyoom status on zephyr/nexus/forge/sentry, check swap usage, zram health, and any recent OOM kills in the kernel log. Produce a summary table with hostname, earlyoom status, swap used/total, zram used/total, and any recent OOM events."
      no_agent: false
---

# Daily OOM Audit (Blueprint)

This skill is a scheduled blueprint. When triggered by cron, it runs `oom-defense` audit steps across all cluster hosts.

## Audit steps

1. Check earlyoom status on each host:
   ```bash
   for h in zephyr nexus forge sentry; do
     echo "$h: $(ssh $h systemctl is-active earlyoom 2>/dev/null || echo unreachable)"
   done
   ```

2. Check swap usage:
   ```bash
   for h in zephyr nexus forge sentry; do
     echo "$h: $(ssh $h free -h | grep Swap)"
   done
   ```

3. Check recent OOM kills:
   ```bash
   for h in zephyr nexus forge sentry; do
     echo "=== $h ==="
     ssh $h "journalctl -k --no-pager 2>/dev/null | grep -i 'oom-kill\|out of memory' | tail -5"
   done
   ```

4. Check zram health:
   ```bash
   for h in zephyr forge; do
     echo "$h: $(ssh $h zramctl 2>/dev/null || echo no zram)"
   done
   ```

## Output format

Produce a Markdown table:

| Host | earlyoom | Swap used/total | Zram used/total | Recent OOM |
|---|---|---|---|---|
| zephyr | active | 0/7.8G | 1.8G/7.8G | none |

If any host has `earlyoom=inactive`, swap > 90%, or recent OOM events, flag them in a WARNING section at the top.
