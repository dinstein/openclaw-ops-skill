---
name: openclaw-ops
description: Use when diagnosing, repairing, or maintaining an OpenClaw Gateway running as a systemd service on the same machine. Covers status checks, config fixes, crash recovery, log analysis, systemd setup, updates, environment validation, and backup/restore.
---

# OpenClaw Operations

You are a rescue agent (Claude Code or secondary OpenClaw instance) operating on a machine running OpenClaw Gateway as a systemd user service. Your job is to diagnose, fix, and maintain the main OpenClaw instance.

**Principle:** Diagnose → Judge → Act → Verify. Never skip steps.

## 1. Status Check

Run these in order to assess overall health:

```bash
# Service status
systemctl --user status openclaw-gateway

# OpenClaw self-check
openclaw status

# Config validation
python3 -c "import json; json.load(open('$HOME/.openclaw/openclaw.json')); print('JSON OK')"

# Port listening (default 18789)
ss -tlnp | grep 18789

# Process alive
pgrep -af openclaw
```

**Interpreting results:**
- `active (running)` + port listening + `openclaw status` clean = healthy
- `inactive/failed` = see §3 Service Restart/Recovery
- Port not listening but process running = config issue (wrong bind address/port)
- `openclaw status` shows errors = see §2 Config Repair

## 2. Configuration Repair

**Config location:** `~/.openclaw/openclaw.json`

### Before any edit

```bash
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak.$(date +%s)
```

### JSON syntax validation

```bash
python3 -c "import json; json.load(open('$HOME/.openclaw/openclaw.json'))"
```

If this fails, the error message shows the line/position. Common causes:
- Trailing comma in arrays/objects
- Missing quotes on keys
- Unescaped characters in strings

### Common config errors

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Channel not connecting | Missing/wrong token in env | Check `~/.openclaw/env` has correct token vars |
| "device identity mismatch" | systemd env token ≠ openclaw.json token | Sync tokens between env file and config |
| Agent not routing | `bindings` field misconfigured | Bindings go at top-level, not inside `agents.list[].routing` |
| Discord bot offline | Wrong guild ID or missing intents | Verify guild ID, check Discord developer portal for intents |

### Validation after fix

```bash
python3 -c "import json; json.load(open('$HOME/.openclaw/openclaw.json')); print('JSON OK')"
openclaw status
```

Then restart (§3) if config changes require it.

## 3. Service Restart & Recovery

### Standard restart

```bash
systemctl --user restart openclaw-gateway
sleep 3
systemctl --user status openclaw-gateway
journalctl --user -u openclaw-gateway -n 20 --no-pager
```

**Always verify** status + recent logs after restart.

### Crash recovery

If the service is in `failed` state:

```bash
# 1. Find out why it crashed
journalctl --user -u openclaw-gateway --since "1 hour ago" --no-pager | grep -iE "error|crash|fatal|SIGTERM|OOM"

# 2. Check resources
free -h
df -h ~

# 3. Fix root cause (config, disk, memory) BEFORE restarting

# 4. Restart
systemctl --user restart openclaw-gateway

# 5. Verify
sleep 5
systemctl --user status openclaw-gateway
```

**Do NOT** blindly restart without checking logs first. Repeated restarts without fixing the root cause waste time.

### Service won't start at all

```bash
# Try manual start for better error output
cd ~/.openclaw && npx openclaw gateway start

# Check Node.js is available
node -v

# Check openclaw is installed
openclaw --version
```

## 4. Log Diagnosis

### Basic log commands

```bash
# Last 50 lines
journalctl --user -u openclaw-gateway -n 50 --no-pager

# Since specific time
journalctl --user -u openclaw-gateway --since "30 min ago" --no-pager

# Follow live
journalctl --user -u openclaw-gateway -f

# Filter errors only
journalctl --user -u openclaw-gateway --no-pager | grep -iE "error|warn|fatal"
```

### Common error patterns

| Log pattern | Meaning | Action |
|------------|---------|--------|
| `EADDRINUSE` | Port already in use | `ss -tlnp \| grep <port>`, kill conflicting process or change port |
| `ECONNREFUSED` | Upstream service unreachable | Check network, Tailscale, API endpoint |
| `Invalid token` / `401` / `403` | Auth failure | Check tokens in `~/.openclaw/env` |
| `ENOMEM` / `JavaScript heap` | Out of memory | Check `free -h`, increase Node heap or free RAM |
| `SyntaxError` in config | Bad JSON | See §2 Config Repair |
| `MODULE_NOT_FOUND` | Missing dependency | `cd ~/.npm-global/lib/node_modules/openclaw && npm install` |

## 5. systemd Initial Setup

For a fresh machine that needs OpenClaw as a persistent service:

### Step 1: Enable linger (persist after logout)

```bash
loginctl enable-linger $(whoami)
```

### Step 2: Create service file

```bash
mkdir -p ~/.config/systemd/user

cat > ~/.config/systemd/user/openclaw-gateway.service << 'EOF'
[Unit]
Description=OpenClaw Gateway
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
EnvironmentFile=%h/.openclaw/env
ExecStart=%h/.npm-global/bin/openclaw gateway start --foreground
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
EOF
```

**Adjust `ExecStart` path** to match actual openclaw binary location (`which openclaw`).

### Step 3: Create env file

```bash
cat > ~/.openclaw/env << 'EOF'
# Add required tokens here, e.g.:
# OPENCLAW_TOKEN=xxx
# DISCORD_TOKEN=xxx
EOF
chmod 600 ~/.openclaw/env
```

### Step 4: Enable and start

```bash
systemctl --user daemon-reload
systemctl --user enable openclaw-gateway
systemctl --user start openclaw-gateway
sleep 3
systemctl --user status openclaw-gateway
```

### Custom environment overrides

For overrides that survive OpenClaw upgrades (which may overwrite the main service file):

```bash
mkdir -p ~/.config/systemd/user/openclaw-gateway.service.d

cat > ~/.config/systemd/user/openclaw-gateway.service.d/env.conf << 'EOF'
[Service]
# Additional env vars or overrides
Environment=NODE_OPTIONS=--max-old-space-size=4096
EOF

systemctl --user daemon-reload
systemctl --user restart openclaw-gateway
```

## 6. Update & Upgrade

### Check for updates

```bash
CURRENT=$(openclaw --version)
LATEST=$(npm view openclaw version)
echo "Current: $CURRENT  Latest: $LATEST"
```

### Perform update

```bash
# 1. Backup config
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.pre-upgrade.$(date +%s)

# 2. Update
npm update -g openclaw

# 3. Verify new version
openclaw --version

# 4. Restart service
systemctl --user restart openclaw-gateway

# 5. Verify
sleep 5
systemctl --user status openclaw-gateway
openclaw status
```

### Rollback

If the new version breaks things:

```bash
# Install specific version
npm install -g openclaw@<previous_version>

# Restore config if needed
cp ~/.openclaw/openclaw.json.pre-upgrade.<timestamp> ~/.openclaw/openclaw.json

# Restart
systemctl --user restart openclaw-gateway
```

## 7. Environment Check

Validate the runtime environment is healthy:

```bash
# Node.js version (requires 18+)
node -v

# npm global prefix
npm config get prefix

# openclaw binary location
which openclaw

# openclaw version
openclaw --version

# Check disk space
df -h ~

# Check memory
free -h

# Check port conflicts
ss -tlnp | grep 18789

# Tailscale (if used)
tailscale status 2>/dev/null || echo "Tailscale not installed/running"
```

### Dependency integrity

If `MODULE_NOT_FOUND` errors appear:

```bash
OPENCLAW_DIR=$(dirname $(dirname $(which openclaw)))
cd "$OPENCLAW_DIR/lib/node_modules/openclaw"
npm install --production
```

## 8. Backup & Restore

### Critical files to back up

```
~/.openclaw/
├── openclaw.json          # Main config (CRITICAL)
├── env                    # Secrets/tokens (CRITICAL)
├── agents/                # Agent configs & auth profiles
├── devices/               # Paired devices
└── workspace/             # Agent workspace (SOUL.md, MEMORY.md, skills, memory/)
```

### Backup

```bash
BACKUP_DIR=~/openclaw-backup-$(date +%Y%m%d-%H%M%S)
mkdir -p "$BACKUP_DIR"

# Config + secrets
cp ~/.openclaw/openclaw.json "$BACKUP_DIR/"
cp ~/.openclaw/env "$BACKUP_DIR/"

# Agent data
cp -r ~/.openclaw/agents "$BACKUP_DIR/"
cp -r ~/.openclaw/devices "$BACKUP_DIR/"

# Workspace (if not git-managed separately)
cp -r ~/.openclaw/workspace "$BACKUP_DIR/"

# systemd service
cp ~/.config/systemd/user/openclaw-gateway.service "$BACKUP_DIR/" 2>/dev/null
cp -r ~/.config/systemd/user/openclaw-gateway.service.d "$BACKUP_DIR/" 2>/dev/null

echo "Backup saved to $BACKUP_DIR"
```

### Restore

```bash
BACKUP_DIR=~/openclaw-backup-<timestamp>

# Stop service first
systemctl --user stop openclaw-gateway

# Restore files
cp "$BACKUP_DIR/openclaw.json" ~/.openclaw/
cp "$BACKUP_DIR/env" ~/.openclaw/
cp -r "$BACKUP_DIR/agents" ~/.openclaw/
cp -r "$BACKUP_DIR/devices" ~/.openclaw/

# Validate config
python3 -c "import json; json.load(open('$HOME/.openclaw/openclaw.json')); print('JSON OK')"

# Restart
systemctl --user start openclaw-gateway
sleep 3
systemctl --user status openclaw-gateway
```

## Safety Rules

1. **Always check logs before changing anything** — understand the problem first
2. **Always backup before editing config** — `cp` with timestamp suffix
3. **Always validate JSON after editing** — one bad comma kills the service
4. **Never print secrets** — check env file exists and has content, don't cat it
5. **Never delete workspace files** — use `trash` if you must remove something
6. **Always verify after restart** — status + logs, don't assume it worked
7. **Destructive operations require confirmation** — ask the user before wiping data

## File Layout Reference

```
~/.openclaw/
├── openclaw.json              # Main config
├── openclaw.json.bak          # Auto-backup
├── env                        # Environment variables (secrets)
├── agents/                    # Per-agent configs
│   └── <agent>/agent/
│       ├── auth-profiles.json
│       └── models.json
├── devices/
│   └── paired.json            # Paired node devices
├── workspace/                 # Agent workspace
│   ├── SOUL.md, USER.md, etc.
│   ├── memory/                # Daily memory logs
│   └── skills/                # Local skills
└── sessions/                  # Session logs

~/.config/systemd/user/
├── openclaw-gateway.service                # Main service file
└── openclaw-gateway.service.d/
    └── env.conf                            # Custom env overrides (survives upgrades)
```
