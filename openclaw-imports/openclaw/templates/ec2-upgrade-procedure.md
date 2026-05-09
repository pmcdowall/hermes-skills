# OpenClaw EC2 Upgrade Procedure

Tested: 2026.4.21 → 2026.4.29 on Ubuntu EC2 (52.63.13.24). May 2026.

## Pre-flight checks

```bash
# 1. Check sessions.json — trim if > 200KB or > 15 entries
ls -lah ~/.openclaw/agents/main/sessions/sessions.json
python3 -c "import json; d=json.load(open('/home/ubuntu/.openclaw/agents/main/sessions/sessions.json')); print(len(d), 'entries')"

# 2. Check WAL files — note sizes
ls -lah ~/.openclaw/flows/*.sqlite* ~/.openclaw/tasks/*.sqlite* 2>/dev/null

# 3. Check child processes
ps aux | grep openclaw | grep -v grep | grep -v uvicorn
```

## Stop gateway (disable first to prevent auto-restart)

```bash
systemctl --user disable openclaw-gateway
systemctl --user stop openclaw-gateway
# Kill any stragglers (TUI, doctor, etc.)
pkill -9 -f openclaw-gateway 2>/dev/null
pkill -9 -f openclaw-tui 2>/dev/null
# Verify clean
ps aux | grep openclaw | grep -v grep | grep -v uvicorn
```

## WAL checkpoint while gateway is stopped

```bash
sqlite3 ~/.openclaw/tasks/runs.sqlite "PRAGMA wal_checkpoint(TRUNCATE);"
sqlite3 ~/.openclaw/flows/registry.sqlite "PRAGMA wal_checkpoint(TRUNCATE);"
# Expected output: 0|0|0 for each — means clean
ls -lah ~/.openclaw/tasks/*.sqlite* ~/.openclaw/flows/*.sqlite* 2>/dev/null
# WAL files should be gone
```

## Install new version

```bash
# MUST use sudo — openclaw is installed at /usr/lib/node_modules (system-owned)
sudo npm install -g openclaw@<TARGET_VERSION>
# Verify
cat /usr/lib/node_modules/openclaw/package.json | python3 -c "import json,sys; d=json.load(sys.stdin); print(d.get('version'))"
/usr/bin/openclaw --version
```

## Fix service entrypoint (ALWAYS required after update)

The systemd service file reliably points to the old binary path after any npm update.

```bash
# Check current (wrong) entrypoint
grep ExecStart ~/.config/systemd/user/openclaw-gateway.service

# Fix it
sed -i "s|ExecStart=.*|ExecStart=/usr/bin/node /usr/lib/node_modules/openclaw/dist/index.js gateway --port 18789|" \
  ~/.config/systemd/user/openclaw-gateway.service

# Verify
grep ExecStart ~/.config/systemd/user/openclaw-gateway.service
# Expected: ExecStart=/usr/bin/node /usr/lib/node_modules/openclaw/dist/index.js gateway --port 18789

systemctl --user daemon-reload
```

## Start gateway and verify

```bash
systemctl --user enable openclaw-gateway
systemctl --user start openclaw-gateway

# Check port — as of 2026.4.29, startup is ~5 seconds (was 135-145s before)
sleep 10
ss -tlnp | grep 18789

# Check runtime status
timeout 30 /usr/bin/openclaw gateway status

# Check channels
timeout 30 /usr/bin/openclaw channels status

# Check logs for errors
journalctl --user -u openclaw-gateway --since "5 minutes ago" --no-pager | \
  grep -E "version|ready|starting|ERROR|error|warn|failed" | head -30
```

## Normal post-startup signals (do NOT panic about these)

- Event loop delay warning in first 30s: P99 up to ~9 seconds during plugin init — normal, settles fast
- `gateway probe` showing "Read probe: failed - timeout": known on this EC2, use `timeout 30 openclaw gateway probe`
- Service PATH warnings about nix-profile, volta, asdf: cosmetic noise, not blocking
- "Service config looks out of date": triggered by PATH warnings above, not a real problem if gateway is running

## Version-specific notes

### 2026.4.29 (upgrade from 2026.4.21)
- Startup dramatically faster (~5s vs 135-145s previously)
- tools.exec no longer implicitly widens messaging/minimal profiles — but this EC2 uses default (full) profile, no impact
- Subagent orphan recovery now automatic — less manual sessions.json surgery needed going forward
- sudo required for npm install (system-owned install path)
