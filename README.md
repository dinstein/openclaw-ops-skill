# openclaw-ops `v1.0.0`

[English](README.md) | [中文](README_CN.md)

A skill that teaches any AI agent how to operate and maintain an [OpenClaw](https://openclaw.ai) Gateway running as a persistent service.

Works with **any agent that has shell access** — Claude Code, Codex, OpenClaw, Pi, or any AI coding agent running on the same machine as the OpenClaw Gateway.

Supports both **Linux (systemd)** and **macOS (launchd)**.

## Install

### From ClawHub

```bash
clawhub install openclaw-ops
```

### Ask your agent to install it

Just tell your agent in a conversation:

```
> Install the openclaw-ops skill from https://github.com/dinstein/openclaw-ops-skill
```

The agent will download `SKILL.md` and place it in the correct skills directory automatically.

### Manual

Copy `SKILL.md` into your agent's skills directory:

```bash
# Claude Code
mkdir -p ~/.claude/skills/openclaw-ops
cp SKILL.md ~/.claude/skills/openclaw-ops/SKILL.md

# OpenClaw
mkdir -p ~/.openclaw/workspace/skills/openclaw-ops
cp SKILL.md ~/.openclaw/workspace/skills/openclaw-ops/SKILL.md

# Other agents — place in whatever skills directory your agent reads from
```

## What it covers

| Module | Description |
|--------|-------------|
| **Status checks** | `openclaw doctor`, service health, port, process |
| **Config repair** | JSON fixing, common misconfiguration patterns, schema validation |
| **Restart & recovery** | Crash diagnosis, safe restart, root cause analysis |
| **Log diagnosis** | journalctl / log file patterns, common error identification |
| **Service setup** | From-scratch systemd / launchd configuration |
| **Updates** | Upgrade, version check, rollback, doctor diff |
| **Environment validation** | Node.js, dependencies, port conflicts, Tailscale |
| **Backup & restore** | Critical files, backup/restore procedures |
| **Session cleanup** | Orphan transcripts, disk management |
| **Health check cron** | Periodic automated monitoring templates |

## Usage Examples

Once installed, you can ask your AI agent to perform ops tasks in natural language:

### Quick health check

```
> Check if my OpenClaw is healthy

Agent runs: openclaw doctor, systemctl status, port check
Agent reports: "Gateway is running, all channels connected, 3 orphan transcripts found — want me to clean them up?"
```

### Diagnose a crash

```
> OpenClaw stopped responding, can you check what happened?

Agent runs: systemctl status (sees 'failed'), checks journalctl for errors
Agent reports: "Gateway crashed 15 minutes ago due to ENOMEM. Your server has 94% memory usage. I can restart it after we free some memory — want me to check what's consuming RAM?"
```

### Fix broken config

```
> I edited openclaw.json and now it won't start

Agent runs: JSON validation (finds trailing comma on line 42), backs up current file, fixes the syntax
Agent reports: "Found a trailing comma on line 42. Fixed it and validated — JSON is clean now. Want me to restart the gateway?"
```

### Set up on a new machine

```
> Set up OpenClaw as a persistent service on this machine

Agent detects OS (Linux/macOS), creates service file (systemd unit or launchd plist),
configures env file, enables linger/auto-start, starts the service, verifies health.
```

### Upgrade safely

```
> Upgrade OpenClaw to the latest version

Agent checks current vs latest version, backs up config, runs npm update,
restarts service, runs doctor before/after diff, reports any new warnings.
```

### Backup before changes

```
> Back up my OpenClaw setup before I make changes

Agent creates timestamped backup of openclaw.json, env, agents/, devices/,
workspace/, and service config. Reports backup location.
```

### Tailscale proxy issues

```
> Dashboard is not accessible via Tailscale

Agent checks: tailscale status, tailscale serve status, localhost connectivity
Agent reports: "Tailscale Serve isn't configured. Want me to set it up to proxy HTTPS to localhost:18789?"
```

### Periodic monitoring

```
> Set up automatic health checks every 6 hours

Agent creates an OpenClaw cron job or system crontab entry that runs
openclaw doctor periodically and reports issues.
```

## How it works

This is a **pure documentation skill** — no scripts, no external dependencies, no framework lock-in. Install it in any agent that can read markdown and run shell commands. It teaches the agent:

1. **What to check** — the right commands and files for each situation
2. **How to interpret** — what the output means and common error patterns
3. **What to do** — step-by-step fix procedures with safety guardrails
4. **How to verify** — confirmation steps after every action

The agent reads the skill when it encounters an ops-related task, then follows the documented procedures using standard shell commands.

## Safety

The skill enforces these rules:

- Always check logs before making changes
- Always backup config before editing
- Always validate JSON after editing
- Never print secrets (env files)
- Never delete workspace files without confirmation
- Always verify after restart

## Versioning

This skill follows [Semantic Versioning](https://semver.org/):

- **MAJOR** — breaking changes to skill structure or safety rules
- **MINOR** — new modules, new platform support, new commands
- **PATCH** — fixes, typos, improved descriptions

Current version: `1.0.0`

## License

MIT
