# OpenClaw Operations Skill

You are operating on a server running OpenClaw as a systemd user service. Your job is to diagnose and fix OpenClaw issues.

## Environment

- **User**: bot
- **OpenClaw workspace**: ~/.openclaw/workspace
- **Config**: ~/.openclaw/openclaw.json
- **Service**: systemctl --user (start|stop|restart|status) openclaw-gateway
- **Logs**: journalctl --user -u openclaw-gateway --no-pager
- **CLI**: openclaw status / openclaw gateway status
- **Node**: v22.22.0
- **OS**: Ubuntu 24.04 LTS

## Quick Diagnosis

Run these in order:

```bash
# 1. Service status
systemctl --user status openclaw-gateway

# 2. Recent logs (last 50 lines)
journalctl --user -u openclaw-gateway -n 50 --no-pager

# 3. OpenClaw self-check
openclaw status

# 4. Config validation
python3 -c "import json; json.load(open('/home/bot/.openclaw/openclaw.json')); print('Config OK')"

# 5. Port check
ss -tlnp | grep 18789

# 6. Process check
pgrep -a openclaw
```

## Common Issues & Fixes

### Service won't start
1. Check logs: `journalctl --user -u openclaw-gateway -n 100 --no-pager`
2. Validate config JSON: `python3 -c "import json; json.load(open('/home/bot/.openclaw/openclaw.json'))"`
3. Check node: `node -v`
4. Try manual start: `cd ~/.openclaw && npx openclaw gateway start`

### Service crashes / restarts repeatedly
1. Look for crash pattern: `journalctl --user -u openclaw-gateway --since "1 hour ago" --no-pager | grep -i "error\|crash\|fatal"`
2. Check memory: `free -h`
3. Check disk: `df -h /home/bot`

### Config issues
- Config location: `~/.openclaw/openclaw.json`
- Env file: `~/.openclaw/env`
- Systemd env override: `~/.config/systemd/user/openclaw-gateway.service.d/env.conf`
- After config edits: `systemctl --user restart openclaw-gateway`

### Channel not working (Discord/Telegram)
1. Check channel config in openclaw.json
2. Look for auth errors: `journalctl --user -u openclaw-gateway --no-pager | grep -i "discord\|telegram\|auth\|token"`
3. Verify tokens in env file

### Update OpenClaw
```bash
npm update -g openclaw
openclaw --version
systemctl --user restart openclaw-gateway
```

## Important Rules

- **Always check logs before changing anything**
- **Backup config before editing**: `cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak`
- **Validate JSON after editing**: `python3 -c "import json; json.load(open('/home/bot/.openclaw/openclaw.json'))"`
- **Never delete workspace files** — use `trash` if removing anything
- **Env file** (~/.openclaw/env) contains secrets — don't print it, just check it exists
- After restart, check: `systemctl --user status openclaw-gateway` and `journalctl --user -u openclaw-gateway -n 20 --no-pager`

## Network / Tailscale

- Tailscale IP: 100.91.15.37
- Gateway binds to loopback, Tailscale Serve proxies HTTPS → localhost:18789
- Dashboard: https://d-ag1-dmit-lax-an5.tail4dcb8c.ts.net/
- Check Tailscale: `tailscale status`

## File Locations

```
~/.openclaw/
├── openclaw.json          # Main config
├── env                    # Environment variables (secrets)
├── workspace/             # Agent workspace (SOUL.md, MEMORY.md, etc.)
├── agents/                # Agent-specific data
└── sessions/              # Session logs

~/.config/systemd/user/
├── openclaw-gateway.service              # Main service (managed by openclaw)
├── openclaw-gateway.service.d/env.conf   # Custom env overrides
└── claude-remote-control.service         # This remote control service
```
