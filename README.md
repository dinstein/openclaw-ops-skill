# openclaw-ops

An [OpenClaw](https://openclaw.ai) skill for operational maintenance of OpenClaw Gateway running as a systemd service.

Designed to be used by a **rescue agent** (Claude Code, secondary OpenClaw instance, or any AI coding agent) running on the same machine as the main OpenClaw Gateway.

## What it covers

- **Status checks** — service health, port, process, config validation
- **Config repair** — JSON fixing, common misconfiguration patterns
- **Restart & recovery** — crash diagnosis, safe restart procedures
- **Log diagnosis** — journalctl patterns, common error identification
- **systemd setup** — from-scratch service configuration with linger
- **Updates** — upgrade, version check, rollback
- **Environment validation** — Node.js, dependencies, port conflicts
- **Backup & restore** — critical files, backup/restore procedures

## Install

### From ClawHub

```bash
clawhub install openclaw-ops
```

### Manual

Copy `SKILL.md` to your agent's skills directory:

```bash
cp SKILL.md ~/.openclaw/workspace/skills/openclaw-ops/SKILL.md
```

## License

MIT
