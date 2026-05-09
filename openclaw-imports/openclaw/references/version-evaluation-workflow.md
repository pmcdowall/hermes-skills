# OpenClaw Version Evaluation Workflow

How to evaluate whether a new OpenClaw release is safe to install on this EC2
(Linux + Telegram channel + bundled plugins only: memory-core, telegram, browser,
file-transfer, ariel agent).

---

## Quick check: is there a new release?

```bash
# npm registry — works from EC2, no browser needed
curl -s "https://registry.npmjs.org/openclaw" | python3 -c "
import json,sys
d=json.load(sys.stdin)
stable=[v for v in d.get('versions',{}).keys() if 'beta' not in v and 'rc' not in v]
latest=sorted(stable)[-1]
print('latest:', latest)
print('recent 5:', sorted(stable)[-5:])
"

# GitHub releases API
curl -s "https://api.github.com/repos/openclaw/openclaw/releases/latest" | python3 -c "
import json,sys; d=json.load(sys.stdin)
print(d.get('tag_name'), d.get('published_at','')[:10])
"
```

---

## Full evaluation sequence (for a new version)

### 1. Fetch release notes

```bash
VERSION="v2026.X.Y"
curl -s "https://api.github.com/repos/openclaw/openclaw/releases/tags/${VERSION}" \
  | python3 -c "import json,sys; d=json.load(sys.stdin); print(d.get('body','')[:6000])"
```

Look for: event loop fixes, agent prep time fixes, Telegram fixes, startup fixes.
Red flags: heartbeat changes, plugin loader changes, undici changes, dispatch changes.

### 2. Check the canary issue (#76562 as of May 2026)

```bash
curl -s "https://api.github.com/repos/openclaw/openclaw/issues/76562" \
  | python3 -c "import json,sys; d=json.load(sys.stdin); print(d.get('state'), d.get('title','')[:80])"
```

If still OPEN → NOT SAFE regardless of other signals.
Issue #76562: "High CPU, extreme control-plane RPC latency, and unstable polling after
upgrade from 2026.4.24" — widespread event-loop saturation in 5.x not yet fixed.

### 3. Search for new regressions specific to this version

```bash
VERSION_SHORT="5.4"  # just the minor.patch for search

curl -s "https://api.github.com/search/issues?q=repo:openclaw/openclaw+is:issue+is:open+${VERSION_SHORT}+regression&sort=updated&per_page=10" \
  | python3 -c "import json,sys; d=json.load(sys.stdin); [print(f'  #{i[\"number\"]} {i[\"title\"][:80]}') for i in d.get('items',[])]"

curl -s "https://api.github.com/search/issues?q=repo:openclaw/openclaw+is:issue+is:open+${VERSION_SHORT}+telegram&sort=updated&per_page=8" \
  | python3 -c "import json,sys; d=json.load(sys.stdin); [print(f'  #{i[\"number\"]} {i[\"title\"][:80]}') for i in d.get('items',[])]"

curl -s "https://api.github.com/search/issues?q=repo:openclaw/openclaw+is:issue+is:open+${VERSION_SHORT}+event+loop&sort=updated&per_page=8" \
  | python3 -c "import json,sys; d=json.load(sys.stdin); [print(f'  #{i[\"number\"]} {i[\"title\"][:80]}') for i in d.get('items',[])]"
```

### 4. Check community confirmation (Reddit blocked from cloud IPs)

Reddit (reddit.com) is BLOCKED from EC2/datacenter IPs. Cannot be fetched directly.
GitHub issues are the primary signal — they are rich enough for evaluation.

---

## Scoring rubric for our setup

SAFE to install when ALL of:
  - Canary issue(s) are CLOSED
  - No new open regression issues affecting: event loop, Telegram, agent prep time,
    gateway startup, TUI deadlock
  - Version is at least 3-5 days old (community has had time to surface problems)

NOT SAFE when ANY of:
  - Canary issue still open
  - New open issues with keywords: event loop, eventLoopDelayMaxMs, agent prep,
    synchronous CPU, gateway hang, Telegram timeout

UNCERTAIN when:
  - Canary closed but new issues just filed and unconfirmed
  - Version released within 24 hours (too fresh)

---

## Automated daily monitoring — cron job

Cron job ID: 7e816f0f17a7 ("OpenClaw Version Monitor")
Schedule: 0 20 * * * UTC (6am AEST daily)
Delivery: Email to paul@onefellswoop.com.au via MS365 Graph API
Logic: silent if no new version; full evaluation + email if new version detected

The cron job checks vs. a hardcoded "last evaluated" version in its prompt.
When a new version is confirmed safe and installed, UPDATE THE CRON JOB PROMPT
to reflect the new "last evaluated" version and current recommended version.

Email delivery uses MS365 Graph API via rocky@onefellswoop.com.au.
Note: op CLI is not on Hermes EC2 — the cron job SSHes to OpenClaw EC2 (52.63.13.24)
to retrieve the 1Password secret. See ofs-platforms skill / references/ms365.md.

---

## Version history quick reference

| Version    | Status             | Key issue                          |
|------------|--------------------|------------------------------------|
| 2026.4.23  | STABLE (use this)  | Clean baseline                     |
| 2026.4.27  | Avoid              | Event loop regression              |
| 2026.4.29  | DO NOT USE         | 4 severe regressions (see notes)   |
| 2026.5.2   | DO NOT USE         | Fixed #75999 but introduced #76562 |
| 2026.5.3   | DO NOT USE         | #78196 extension plugin silent skip|
| 2026.5.3-1 | DO NOT USE         | Hotfix for 5.3 only                |
| 2026.5.4   | HOLD               | #76562 still open (May 2026)       |

Full version stability notes: references/version-stability-may2026.md
