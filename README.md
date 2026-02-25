# openclaw-ops-skill

Claude Code skill for OpenClaw server operations and troubleshooting.

## What is this?

A Claude Code skill (`.claude/skills/`) that gives Claude Code the knowledge to diagnose and fix OpenClaw issues remotely. Designed to work with Claude Code Remote Control — when OpenClaw is down, connect from your phone and let Claude Code fix it.

## Features

- Quick 6-step diagnosis checklist
- Common issue fixes (service crash, config errors, channel failures)
- All critical file paths and commands
- Safety rules (backup before edit, validate JSON, never print secrets)
- Network/Tailscale info

## Install

Copy `SKILL.md` to your OpenClaw server:

```bash
mkdir -p ~/.openclaw/.claude/skills/openclaw-ops
cp SKILL.md ~/.openclaw/.claude/skills/openclaw-ops/
```

## Usage

Start Claude Code Remote Control on your server:

```bash
claude remote-control
```

Connect from phone/browser, then say:
- "OpenClaw is down, help me fix it"
- "Check OpenClaw logs for errors"
- "Restart OpenClaw and verify it's working"

## License

MIT
