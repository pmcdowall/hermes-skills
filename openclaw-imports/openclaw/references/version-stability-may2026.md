# OpenClaw Version Stability Notes — May 2026

## Recommended stable version: 2026.4.23

---

## 2026.5.4 — DO NOT UPGRADE YET (as of May 6, 2026)

Published May 5, 2026. Too fresh (1 day old). The 5.x line has active
instability reports not addressed in 5.4. Hold on 4.23.

### Key unresolved issues in 5.x line

**#76562 — High CPU, extreme control-plane RPC latency (STILL OPEN)**
Multiple users on 5.2-5.4 reporting gateway-looks-healthy-but-is-dying pattern:
- Control-plane methods: sessions.usage ~843s, config.get ~204s, cron.list ~144s
- Widespread event-loop saturation despite gateway reporting "active"
- Downgrading binary alone does NOT fix it — config/state must also be reverted
- As of May 3, main branch has only "partial mitigations for sessions.list and
  models.list, not a proven fix for the broader cron, node, Telegram, config-state behavior"
- 5.4 release notes do NOT claim to fix this issue
- This is the upgrade canary: wait until #76562 is closed before upgrading

**#75844 — TUI fatal deadlock (STILL OPEN)**
Not fixed in 5.x. Low impact for our setup (TUI rarely used for live work).

**#75539 — Telegram IPv6 event loop blocking (STILL OPEN, partially mitigated)**
5.2 improved Telegram-specific paths (process DNS order inherited, sticky IPv4
fallback quieter). Our EC2 already had the sticky IPv4 workaround. Residual
risk is low for EC2/Linux but issue is not formally closed.

**#78007 — fetch() TypeError in 5.4 for external channel plugins (OPEN)**
Specific to third-party external plugins (WeChat etc). Bundled plugins unaffected.

**#78196 — Extension plugins silently skipped by gateway loader in 5.3+ (OPEN)**
Affects custom extension plugins (plugins.load.paths). Not our use case.

### What 5.x did fix

**#75999 — Agent prep 73-100s event loop blocking: FIXED in 5.2**
Closed May 2, 2026. System-prompt cache, plugin tool prep cache, and heartbeat
runaway fix (was firing every ~10s instead of every 30m, causing >6000ms
event loop spikes).

**#75824 — OpenAI transport silent reply drops: FIXED**
Not showing in open issues.

### Re-evaluate when

- #76562 is closed with a confirmed fix, OR
- 10-14 days pass and community confirms 5.4/5.5 is stable on Telegram+EC2 setups

---

## 2026.5.3-1 — DO NOT USE

Hotfix for 5.3 plugin security scanner. Inherits all 5.3 and 5.2 issues.

## 2026.5.3 — DO NOT USE

Introduced extension plugin silent-skipping regression (#78196).

## 2026.5.2 — DO NOT USE

First 5.x stable release. Fixed #75999 (the big event loop killer) but
introduced #76562 (control-plane saturation). Users on 5.2 still reporting
extreme latency in Telegram context as of May 3.

---

## 2026.4.29 — DO NOT USE (severe regressions)

**#75999 / #75290 — Agent prep 73-100 seconds per turn**
Every message dispatch blocks the event loop for 34+ seconds synchronously.
Result: agent times out before producing any reply; web UI bubble disappears silently.

**#75824 — Web UI chat bubble disappears, no reply (OpenAI transport)**
Specific to OpenAI primary model. Fix: switch primary model to anthropic/claude-sonnet-4-6.

**#75844 / #75379 / #75717 — TUI fatal deadlock**
TUI connects at TCP level but hangs before/after auth. Multiple root causes.
Workaround: OPENCLAW_NO_RESPAWN=1 partially helps but doesn't fully fix.

**#75539 — Telegram/undici IPv6-first on servers without global IPv6**
Bundled undici v8.1.0 tries IPv6 first, blocks event loop 10-31s per Telegram call.
Gateway self-recovers with "sticky IPv4-only dispatcher" but startup spike remains.

**#75759 — Idle gateway burns 14% CPU**
Main event loop thread in Runnable state 24/7, causes cascading Telegram timeouts.

---

## 2026.4.23 — Stable (CURRENT RECOMMENDED)

Clean version. Agent prep is fast (< 5s). TUI connects and works.
Telegram starts cleanly. No silent no-reply bugs.
Config warning "last written by 2026.4.29" is harmless — 4.23 ignores unknown fields.

---

## 2026.4.27 — Has issues

Multiple reporters confirmed same event-loop blocking as 4.29 (just slightly less severe).
Not recommended as a rollback target.

---

## 2026.4.15 — Previously stable (older)

Pre-dates the dispatch rework. Used as fallback in earlier troubleshooting.

---

## Downgrade Procedure (EC2)

1. systemctl --user disable openclaw-gateway
2. systemctl --user stop openclaw-gateway
3. pkill -9 -f openclaw-gateway 2>/dev/null
4. sqlite3 ~/.openclaw/tasks/runs.sqlite "PRAGMA wal_checkpoint(TRUNCATE);"
5. sqlite3 ~/.openclaw/flows/registry.sqlite "PRAGMA wal_checkpoint(TRUNCATE);"
6. sudo npm install -g openclaw@2026.4.23
7. grep ExecStart ~/.config/systemd/user/openclaw-gateway.service
   # Fix if needed: sed -i "s|ExecStart=.*|ExecStart=/usr/bin/node /usr/lib/node_modules/openclaw/dist/index.js gateway --port 18789|" ~/.config/systemd/user/openclaw-gateway.service
8. systemctl --user daemon-reload
9. openclaw --version  # confirm 2026.4.23
10. systemctl --user enable openclaw-gateway && systemctl --user start openclaw-gateway

## Downgrade Procedure (Mac, homebrew)

1. launchctl stop ai.openclaw.node
2. launchctl stop ai.openclaw.node-host
3. /opt/homebrew/bin/npm install -g openclaw@2026.4.23
4. /opt/homebrew/bin/openclaw --version  # confirm 2026.4.23
5. launchctl start ai.openclaw.node
6. launchctl start ai.openclaw.node-host

Note: if ServBay is installed, its node takes PATH precedence over homebrew.
Check: which -a openclaw (in a login shell with bash -l -c)
Fix: /Applications/ServBay/package/node/current/bin/npm uninstall -g openclaw
Then homebrew copy wins automatically.

## Checking for new releases

npm registry (no browser needed, works from EC2):
  curl -s "https://registry.npmjs.org/openclaw" | python3 -c "
  import json,sys; d=json.load(sys.stdin)
  print('tags:', d.get('dist-tags',{}))
  stable=[v for v in d.get('versions',{}).keys() if 'beta' not in v]
  print('latest stable versions:', sorted(stable)[-5:])
  "

GitHub issues API (Reddit is blocked from cloud/datacenter IPs):
  # Search for specific terms:
  curl -s "https://api.github.com/search/issues?q=TERM+repo:openclaw/openclaw&sort=updated&per_page=10"
  # List recent open issues:
  curl -s "https://api.github.com/repos/openclaw/openclaw/issues?state=open&sort=updated&per_page=30"
  # Get full issue + comments:
  curl -s "https://api.github.com/repos/openclaw/openclaw/issues/NNNN"
  curl -s "https://api.github.com/repos/openclaw/openclaw/issues/NNNN/comments"
  # Check commits ahead of a tag:
  curl -s "https://api.github.com/repos/openclaw/openclaw/compare/2026.4.23...main"
