---
name: openclaw-ops
version: 1.0.0
description: Use when diagnosing, repairing, or maintaining an OpenClaw Gateway on the same machine. Supports both Linux (systemd) and macOS (launchd). Covers status checks, config fixes, crash recovery, log analysis, service setup, updates, environment validation, and backup/restore.
repository: https://github.com/dinstein/openclaw-ops-skill
---

# OpenClaw Operations

You are an AI agent with shell access, operating on a machine running OpenClaw Gateway as a persistent service. Your job is to diagnose, fix, and maintain the OpenClaw Gateway instance on this machine.

**Principle:** Diagnose → Judge → Act → Verify. Never skip steps.

## Platform Detection

Detect the platform first — commands differ between Linux (systemd) and macOS (launchd):

```bash
OS=$(uname -s)  # "Linux" or "Darwin"
echo "Platform: $OS"
```

Use the platform-appropriate commands throughout this guide. Both are provided for each module.

## Port Detection

Do NOT assume port 18789. Detect the actual configured port first:

```bash
# Preferred: read from config
PORT=$(openclaw config get gateway.port 2>/dev/null | grep -oE '[0-9]+')
PORT=${PORT:-18789}  # fallback to default only if config unavailable
echo "Gateway port: $PORT"
```

Use `$PORT` in all port-related commands throughout this guide.

## 1. Status Check

### Quick diagnosis (cross-platform)

```bash
# Comprehensive health check — run this FIRST
openclaw doctor

# Lighter status check
openclaw status

# Config validation
python3 -c "import json; json.load(open('$HOME/.openclaw/openclaw.json')); print('JSON OK')"

# Process alive
pgrep -af openclaw
```

`openclaw doctor` checks:
- **Legacy state** — orphan session files, stale keys
- **State integrity** — orphan transcripts consuming disk
- **Session locks** — stale locks from crashed processes
- **Other gateway services** — conflicting services on the same machine
- **Skills status** — eligible, missing requirements, blocked
- **Plugins** — loaded, disabled, errors
- **Channels** — connectivity test (Telegram, Discord, etc.)
- **Agents** — registered agents list
- **Channel warnings** — config issues (e.g. non-numeric guild channel IDs)

If `openclaw doctor` reports fixable issues, run `openclaw doctor --fix` to auto-repair.

### Service-level check

**Linux (systemd):**
```bash
systemctl --user status openclaw-gateway
ss -tlnp | grep $PORT
```

**macOS (launchd):**
```bash
launchctl list | grep openclaw
lsof -iTCP:$PORT -sTCP:LISTEN
```

**Interpreting results:**
- Service running + port listening + `openclaw doctor` clean = healthy
- Service not running = see §3 Service Restart/Recovery
- Port not listening but process running = config issue (wrong bind address/port)
- `openclaw doctor` shows errors = fix them first, then restart if needed

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
| Channel not connecting | Missing/wrong token in env | Check env file has correct token vars |
| "device identity mismatch" | Service env token ≠ openclaw.json token | Sync tokens between env file and config |
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

**Linux:**
```bash
systemctl --user restart openclaw-gateway
sleep 3
systemctl --user status openclaw-gateway
journalctl --user -u openclaw-gateway -n 20 --no-pager
```

**macOS:**
```bash
PLIST_LABEL="com.openclaw.gateway"  # adjust if different
launchctl kickstart -k "gui/$(id -u)/$PLIST_LABEL"
sleep 3
launchctl list | grep openclaw
# Check log file (see §4 for log location)
```

**Always verify** status + recent logs after restart.

### Crash recovery

**Linux:**
```bash
# 1. Find out why it crashed
journalctl --user -u openclaw-gateway --since "1 hour ago" --no-pager | grep -iE "error|crash|fatal|SIGTERM|OOM"

# 2. Check resources
free -h
df -h ~

# 3. Fix root cause BEFORE restarting
# 4. Restart and verify (see above)
```

**macOS:**
```bash
# 1. Check crash logs
log show --predicate 'process == "node"' --last 1h | grep -iE "error|crash|fatal"
# Also check the log file if configured in plist (see §4)

# 2. Check resources
vm_stat | head -5
df -h ~

# 3. Fix root cause BEFORE restarting
```

**Do NOT** blindly restart without checking logs first.

### Service won't start at all

```bash
# Check Node.js is available
node -v

# Check openclaw is installed and find its path
which openclaw
openclaw --version

# Try manual start for better error output (cross-platform)
openclaw gateway start
# If 'openclaw' not in PATH, use full path:
# $(npm root -g)/openclaw/bin/openclaw.mjs gateway start
```

## 4. Log Diagnosis

### Linux (journalctl)

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

### macOS (log files)

macOS launchd logs go to the files specified in the plist (`StandardOutPath` / `StandardErrorPath`):

```bash
# Default log locations (adjust to match your plist)
LOG_DIR="$HOME/.openclaw/logs"
tail -50 "$LOG_DIR/gateway.log"
tail -50 "$LOG_DIR/gateway-error.log"

# Follow live
tail -f "$LOG_DIR/gateway.log"

# Filter errors
grep -iE "error|warn|fatal" "$LOG_DIR/gateway.log"

# macOS unified log (if process logs there)
log show --predicate 'process == "node"' --last 30m
```

### Common error patterns (both platforms)

| Log pattern | Meaning | Action |
|------------|---------|--------|
| `EADDRINUSE` | Port already in use | Find conflicting process and kill or change port |
| `ECONNREFUSED` | Upstream unreachable | Check network, Tailscale, API endpoint |
| `Invalid token` / `401` / `403` | Auth failure | Check tokens in env file |
| `ENOMEM` / `JavaScript heap` | Out of memory | Check memory, increase Node heap |
| `SyntaxError` in config | Bad JSON | See §2 Config Repair |
| `MODULE_NOT_FOUND` | Missing dependency | Reinstall openclaw dependencies |

**Port conflict resolution:**

```bash
# Linux
ss -tlnp | grep <port>

# macOS
lsof -iTCP:<port> -sTCP:LISTEN
```

## 5. Update & Upgrade

### Check for updates (cross-platform)

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
```

**Linux — restart:**
```bash
systemctl --user restart openclaw-gateway
sleep 5
systemctl --user status openclaw-gateway
```

**macOS — restart:**
```bash
launchctl kickstart -k "gui/$(id -u)/com.openclaw.gateway"
sleep 5
launchctl list | grep openclaw
tail -20 ~/.openclaw/logs/gateway.log
```

Then run `openclaw status` to verify.

### Rollback

```bash
npm install -g openclaw@<previous_version>
# Restore config if needed
cp ~/.openclaw/openclaw.json.pre-upgrade.<timestamp> ~/.openclaw/openclaw.json
# Restart (use platform-appropriate command above)
```

## 6. Environment Check

```bash
# Cross-platform checks
node -v                    # Requires 18+
which openclaw
openclaw --version
df -h ~

# Linux-specific
free -h
ss -tlnp | grep $PORT
tailscale status 2>/dev/null

# macOS-specific
vm_stat | head -5
lsof -iTCP:$PORT -sTCP:LISTEN
/Applications/Tailscale.app/Contents/MacOS/Tailscale status 2>/dev/null || tailscale status 2>/dev/null
```

### npm path issues (macOS)

If `openclaw` not found after install:

```bash
# Check global prefix
npm config get prefix

# Homebrew Node
export PATH="/opt/homebrew/bin:$PATH"

# Or npm global bin
export PATH="$(npm config get prefix)/bin:$PATH"
```

### Dependency integrity

If `MODULE_NOT_FOUND` errors appear:

```bash
OPENCLAW_DIR=$(npm root -g)/openclaw
cd "$OPENCLAW_DIR"
npm install --production
```

## 7. Backup & Restore

### Critical files to back up

```
~/.openclaw/
├── openclaw.json          # Main config (CRITICAL)
├── env                    # Secrets/tokens (CRITICAL)
├── agents/                # Agent configs & auth profiles
├── devices/               # Paired devices
└── workspace/             # Agent workspace

# Linux only:
~/.config/systemd/user/
├── openclaw-gateway.service
└── openclaw-gateway.service.d/

# macOS only:
~/Library/LaunchAgents/com.openclaw.gateway.plist
```

### Backup

```bash
BACKUP_DIR=~/openclaw-backup-$(date +%Y%m%d-%H%M%S)
mkdir -p "$BACKUP_DIR"

# Core (cross-platform)
cp ~/.openclaw/openclaw.json "$BACKUP_DIR/"
[ -f ~/.openclaw/env ] && cp ~/.openclaw/env "$BACKUP_DIR/" || echo "No env file (tokens may be in systemd drop-in or plist)"
cp -r ~/.openclaw/agents "$BACKUP_DIR/"
cp -r ~/.openclaw/devices "$BACKUP_DIR/"
cp -r ~/.openclaw/workspace "$BACKUP_DIR/"

# Service config
if [ "$(uname -s)" = "Linux" ]; then
    cp ~/.config/systemd/user/openclaw-gateway.service "$BACKUP_DIR/" 2>/dev/null
    cp -r ~/.config/systemd/user/openclaw-gateway.service.d "$BACKUP_DIR/" 2>/dev/null
elif [ "$(uname -s)" = "Darwin" ]; then
    cp ~/Library/LaunchAgents/com.openclaw.gateway.plist "$BACKUP_DIR/" 2>/dev/null
fi

echo "Backup saved to $BACKUP_DIR"
```

### Restore

```bash
BACKUP_DIR=~/openclaw-backup-<timestamp>

# Stop service
if [ "$(uname -s)" = "Linux" ]; then
    systemctl --user stop openclaw-gateway
else
    launchctl unload ~/Library/LaunchAgents/com.openclaw.gateway.plist 2>/dev/null
fi

# Restore files
cp "$BACKUP_DIR/openclaw.json" ~/.openclaw/
cp "$BACKUP_DIR/env" ~/.openclaw/
cp -r "$BACKUP_DIR/agents" ~/.openclaw/
cp -r "$BACKUP_DIR/devices" ~/.openclaw/

# Validate
python3 -c "import json; json.load(open('$HOME/.openclaw/openclaw.json')); print('JSON OK')"

# Start service
if [ "$(uname -s)" = "Linux" ]; then
    systemctl --user start openclaw-gateway
else
    launchctl load ~/Library/LaunchAgents/com.openclaw.gateway.plist
fi

sleep 3
openclaw status
```

## 8. Troubleshooting Quick Index

| Symptom | Start here |
|---------|-----------|
| Gateway won't start | §3 → §4 → §6 |
| Channel disconnected (Discord/Telegram) | §2 (config) → §4 (logs) |
| Config broken after edit | §2 (repair + schema) |
| Disk filling up | §8a Session Cleanup |
| After upgrade something broke | §5 (rollback) → §8b Doctor Diff |
| Tailscale not proxying | §8c Tailscale Serve |
| Need periodic health monitoring | §8d Health Check Cron |

### 8a. Session & Transcript Cleanup

`openclaw doctor` may report orphan transcripts consuming disk:

```bash
# Check disk usage of session data
du -sh ~/.openclaw/agents/*/sessions/

# Doctor reports orphans — auto-fix them
openclaw doctor --fix

# Manual cleanup: remove transcripts older than 30 days
find ~/.openclaw/agents/*/sessions/ -name "*.jsonl" -mtime +30 -exec ls -lh {} \;
# Review the list, then delete if safe:
find ~/.openclaw/agents/*/sessions/ -name "*.jsonl" -mtime +30 -delete
```

**Note:** `openclaw doctor --fix` only removes files it confirms are orphaned (not referenced by `sessions.json`). It's safe to run.

For multi-agent setups, check each agent directory:
```bash
for agent_dir in ~/.openclaw/agents/*/; do
    agent=$(basename "$agent_dir")
    size=$(du -sh "$agent_dir/sessions/" 2>/dev/null | cut -f1)
    count=$(find "$agent_dir/sessions/" -name "*.jsonl" 2>/dev/null | wc -l)
    echo "$agent: $size ($count transcripts)"
done
```

### 8b. Doctor Diff (Before/After Upgrade)

Capture `openclaw doctor` output before and after upgrades to spot regressions:

```bash
# Before upgrade — save baseline
openclaw doctor 2>&1 | tee /tmp/doctor-before.txt

# ... perform upgrade (§5) ...

# After upgrade — compare
openclaw doctor 2>&1 | tee /tmp/doctor-after.txt
diff /tmp/doctor-before.txt /tmp/doctor-after.txt
```

Look for:
- New warnings or errors that didn't exist before
- Changed plugin/skill counts (something got disabled?)
- New "legacy state" detections
- Channel connectivity changes

### 8c. Tailscale Serve Troubleshooting

If OpenClaw uses Tailscale Serve as a reverse proxy (HTTPS → localhost):

```bash
# Check Tailscale status
tailscale status

# Check what Serve is proxying
tailscale serve status

# Verify the proxy target is reachable
curl -s -o /dev/null -w "%{http_code}" http://localhost:$PORT/healthz || echo "Gateway not reachable on localhost"

# Common issues:
# 1. Tailscale not running → systemctl start tailscaled (Linux) or open Tailscale app (macOS)
# 2. Serve not configured → tailscale serve https / $PORT
# 3. Gateway not listening → see §1 Status Check
# 4. Funnel vs Serve confusion → Serve = tailnet only, Funnel = public internet
```

**Reconfigure Tailscale Serve:**
```bash
# Reset and reconfigure
tailscale serve reset
tailscale serve https / http://localhost:$PORT
tailscale serve status
```

### 8d. Health Check Cron Template

Set up periodic health monitoring using OpenClaw's cron system. Use this from your main OpenClaw agent (not the rescue agent):

**Via the `cron` tool (from inside OpenClaw):**
```json
{
  "name": "openclaw-healthcheck",
  "schedule": { "kind": "cron", "expr": "0 */6 * * *", "tz": "UTC" },
  "payload": {
    "kind": "agentTurn",
    "message": "Run openclaw doctor and openclaw status. If any issues found, summarize them. If all healthy, reply with a one-line status.",
    "timeoutSeconds": 120
  },
  "sessionTarget": "isolated",
  "delivery": { "mode": "announce" }
}
```

This runs every 6 hours in an isolated session and announces results to the chat.

**From the rescue agent (via CLI):**
```bash
# Simple crontab-based check (add via crontab -e)
# Runs every 6 hours, logs output
0 */6 * * * /usr/local/bin/openclaw doctor --non-interactive >> ~/.openclaw/logs/doctor-cron.log 2>&1
```

## 9. `openclaw doctor` Reference

### Flags

| Flag | Effect |
|------|--------|
| (none) | Interactive health check with prompts |
| `--fix` | Apply recommended repairs (safe migrations, orphan cleanup) |
| `--force` | Aggressive repairs — may overwrite custom service config |
| `--deep` | Scan system services for extra gateway installs |
| `--non-interactive` | No prompts, safe migrations only (good for cron) |
| `--generate-gateway-token` | Generate and configure a gateway token |
| `--no-workspace-suggestions` | Skip workspace memory system suggestions |

### What `--fix` repairs

- Canonicalizes legacy session keys
- Removes orphan transcript files (not referenced by `sessions.json`)
- Cleans stale session lock files (from crashed processes)
- Applies safe state migrations

**`--fix` does NOT:**
- Modify `openclaw.json` config
- Change service files (unless `--force`)
- Delete workspace files

### Config Schema Validation

Beyond JSON syntax, validate config structure against OpenClaw's schema:

```bash
# Get current config
openclaw config get

# Check specific values
openclaw config get gateway.bind
openclaw config get channels.discord

# Set a value
openclaw config set gateway.bind loopback

# The gateway tool also validates schema on config.apply
```

### `openclaw gateway` vs `openclaw status` vs `openclaw doctor`

| Command | Purpose |
|---------|---------|
| `openclaw status` | Quick status: is gateway running, what version, basic info |
| `openclaw doctor` | Deep health check: state integrity, channels, plugins, skills, warnings |
| `openclaw doctor --fix` | Health check + auto-repair safe issues |
| `openclaw gateway status` | Gateway daemon status (start/stop/restart controls) |
| `openclaw gateway start` | Start gateway in foreground (useful for debugging) |

## Safety Rules

1. **Always check logs before changing anything** — understand the problem first
2. **Always backup before editing config** — `cp` with timestamp suffix
3. **Always validate JSON after editing** — one bad comma kills the service
4. **Never print secrets** — check env file exists and has content, don't cat it
5. **Never delete workspace files** — use `trash` if you must remove something
6. **Always verify after restart** — status + logs, don't assume it worked
7. **Destructive operations require confirmation** — ask the user before wiping data

## Common Interaction Patterns

When a user asks for help, follow these patterns:

**"Is my OpenClaw OK?"** → Run §1 Status Check → report results, offer to fix issues

**"It's not working"** → Run §1 → if service down, §4 Log Diagnosis → §3 Restart after fixing root cause

**"I changed the config and broke it"** → §2 Config Repair → validate → restart if needed

**"Set it up from scratch"** → Detect platform → `openclaw daemon install` → §1 verify

**"Update it"** → §8b Doctor Diff (baseline) → §5 Update → §8b (compare) → verify

**"Clean up disk space"** → §8a Session Cleanup → report freed space

**General debugging flow:**
1. `openclaw doctor` — get the big picture
2. Logs — find the specific error
3. Fix — address root cause
4. Restart — bring service back
5. Verify — confirm it's healthy

## File Layout Reference

```
~/.openclaw/
├── openclaw.json              # Main config
├── openclaw.json.bak          # Auto-backup
├── env                        # Environment variables (secrets)
├── logs/                      # macOS: launchd log output
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

# Linux service config:
~/.config/systemd/user/
├── openclaw-gateway.service
└── openclaw-gateway.service.d/
    └── env.conf               # Custom env overrides

# macOS service config:
~/Library/LaunchAgents/
└── com.openclaw.gateway.plist
```
