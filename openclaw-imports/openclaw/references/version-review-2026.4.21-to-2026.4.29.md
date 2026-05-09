# OpenClaw Version Review: 2026.4.21 → 2026.4.29

Reviewed: 2026-05-01. EC2 install: 2026.4.21 at /usr/lib/node_modules/openclaw.
Node: v24.15.0 (meets >=22.14.0 requirement). No Node upgrade needed.

## Breaking Changes

### tools.exec / tools.fs no longer widen restrictive profiles
Config path: tools.profile = "messaging" | "minimal"
Impact: Users on restrictive profiles who relied on implicit exec/fs widening must add
  explicit `alsoAllow` entries. Startup warning emitted on affected configs.
Paul's config: tools.profile not set (default "full") → NOT AFFECTED.
  tools section at time of review:
    { "web": {"search": {"enabled": true, "provider": "gemini"}},
      "exec": {"host": "auto"},
      "sessions": {"visibility": "all"},
      "agentToAgent": {"enabled": true, "allow": ["main", "ariel"]} }

## Positive Changes Relevant to This Install

### Gateway/systemd exit-78 (EADDRINUSE / lock conflicts)
Gateway now exits with code 78 on port conflict or lock conflict, enabling
RestartPreventExitStatus=78 to stop restart loops. Relevant to past EADDRINUSE issues.

### Automatic orphan/wedged-session recovery
Gateway now auto-tombstones wedged subagent sessions; doctor reconciles them.
Reduces need for manual sessions.json surgery (documented extensively in openclaw skill).
Does NOT eliminate the need to keep sessions.json trimmed — pre-trim still recommended.

### Plugin runtime-deps staging fixes
Multiple fixes for stale symlinks, incomplete AJV/MCP SDK installs, and pnpm relaunch
on gateway startup. Relevant to 6 active plugins: acpx, browser, device-pair,
phone-control, talk-voice, whatsapp.
- Mirrored root chunks now refreshed via temp file (prevents deleting chunks in active use)
- Staged package entry verified before reusing mirrors (fixes browser-control post-update)
- Stale symlinked mirror target roots replaced before writing temp files (prevents crash-loop
  on cross-version container upgrades)

### Cron: disabled job schedule validation
Invalid cron schedule edits on disabled jobs no longer partially mutate stored jobs.
Relevant: several cron jobs on this install.

## Notable Fixes (not directly breaking)

- CLI/agents/status: openclaw agents, agents list, status now on read-only paths — no longer
  preloads plugin runtimes. Reduces child process spawning on status calls (relevant to
  Rocky Dashboard server.js anti-pattern documented in skill).
- Agents/local models: context window guard now uses effective model window, not fixed 16k/32k.
- Security/outbound: HTML tag sanitization improvement.
- Security/secrets: timing-safe buffer comparison for credentials.
- Gateway/sessions: abort RPCs now wait for sessions to actually halt before resolving.
- Bonjour/mDNS: flapping advertiser restart now capped in sliding window (constrained hosts).

## Changelog Review Technique Used

  curl -fsSL https://raw.githubusercontent.com/openclaw/openclaw/main/CHANGELOG.md | \
    grep '^## '   # list all version anchors

  curl -fsSL https://raw.githubusercontent.com/openclaw/openclaw/main/CHANGELOG.md | \
    awk '/^## 2026\.4\.29/,/^## 2026\.1\.10/'   # extract target section

  curl -fsSL https://raw.githubusercontent.com/openclaw/openclaw/main/CHANGELOG.md | \
    awk '/^## 2026\.4\.29/,/^## 2026\.1\.10/' | grep -A 300 '^### Fixes'   # just fixes

  curl -fsSL https://raw.githubusercontent.com/openclaw/openclaw/main/package.json | \
    grep -i 'engines\|node'   # check Node requirement

## Upgrade Recommendation

No blocking issues. Safe to upgrade. Follow the pre-upgrade ritual in the skill:
stop gateway → trim sessions.json if needed → npm install → doctor → start.
Budget 2-3 minutes for gateway startup after install.
After first boot, check: openclaw logs | grep -i 'warn\|alsoAllow\|profile'
