# OpenClaw Version Stability Notes

Last updated: 2026-05-02

## Current Production Version on Paul's EC2: 2026.4.23

### Why 2026.4.29 was abandoned (confirmed May 2026)

Multiple concurrent regressions in 2026.4.29 made it unusable on this EC2 setup.
All confirmed via GitHub issues and live reproduction.

**#75999 / #75290 — Agent prep 73-100 seconds per turn (BLOCKING)**
Every message dispatch blocks the event loop for 34+ seconds. Stage breakdown from
real traces: system-prompt 23s, stream-setup 20s, core-plugin-tools 9-19s. Agent
turns time out before completing. Root cause: synchronous CPU work in dispatch prep
path, substantially reworked between 4.27 and 4.29. Fixed on main, not released.

**#75824 — Web UI chat bubble disappears, no reply (OpenAI transport)**
With OpenAI as primary model, the Responses WebSocket stream path silently fails to
surface replies. OpenAI platform shows tokens consumed but OpenClaw records
assistantTexts: [] and idleTimedOut: true. Workaround: switch to Anthropic.
Not fixed in 4.29 release.

**#75844 / #75379 / #75717 — TUI fatal deadlock**
TUI connects at TCP level, gateway sends auth challenge, TUI event loop blocks before
responding. Multiple root causes: stale dist files after npm update, model normalization
on cold start (100% CPU), drainInput with no timeout bounds. Fixed on main, not released.

**#75539 — Telegram/undici IPv6-first blocking event loop**
undici v8.1.0 (bundled in plugin-runtime-deps) uses happy-eyeballs IPv6-first strategy.
On EC2 (IPv4-only external), every Telegram API call tries IPv6 first, times out after
10-20s, then falls back. Causes eventLoopDelayMaxMs=31000+ on startup. Gateway auto-enables
sticky IPv4-only dispatcher after first failure, but startup window is brutal.

**Summary of fixes on main (not yet released as of 2026-05-02):**
- ACPX runtime deferred (loads only when actually used) — fixes 73s prep
- resolvePluginTools scoped with activate:false, toolDiscovery:true — fixes core-plugin-tools
- Gateway registry reuse — avoids fresh hot-path plugin loads per run
- Async session transcript reads — fixes event loop blocking from fs.readFileSync

### 2026.4.27 — Also problematic, not recommended

Confirmed by multiple reporters to have the same sync prep-stage issues as 4.29.
One workaround circulated: pin to 4.27 + OPENCLAW_ALLOW_OLDER_BINARY_DESTRUCTIVE_ACTIONS=1
+ strip plugins.entries.active-memory.config. Not recommended — too fragile.

### 2026.4.23 — Current stable choice

- No prep-stage regression (predates 4.24 dispatch flow rework)
- TUI connects and accepts input reliably (both EC2 local and Mac via tunnel)
- Web UI responds correctly with Anthropic as primary model
- Gateway starts in ~7 seconds (vs 37-145s on 4.29)
- Config written by 4.29 still loads — "Config was last written by newer version" warning
  is cosmetic and harmless

### 2026.4.21 — Previous Mac homebrew version

Was shadowed by ServBay's 4.29 install (ServBay first in PATH).
After ServBay removal + upgrade to 4.23 via homebrew, Mac CLI is now consistent with EC2.

### When a new version drops

Check CHANGELOG.md before upgrading:
  curl -fsSL https://raw.githubusercontent.com/openclaw/openclaw/main/CHANGELOG.md | \
    grep '^## ' | head -20

Key things to check:
- Any new "prep stages" or "dispatch flow" changes (risk of perf regression)
- Agent session / sessions.json changes
- Plugin runtime-deps changes
- Node.js engine requirement (check engines.node in package.json on the tag)
- Whether the GitHub issues for #75999, #75824, #75844 are closed with a release tag

A new release, **v2026.5.02**, has been spotted (reported by Paul, May 2026). This is a
major version jump (4.x → 5.x) — not the incremental 4.30 that was expected. The jump
is both promising (likely includes the main-branch fixes) and risky (bigger surface area,
possible new regressions, possible schema changes). **Do not deploy without checking:**
- CHANGELOG — are #75999, #75844, #75824, #75539 listed as fixed?
- Node.js requirement — still compatible with Node 24 on EC2?
- sessions.json / SQLite schema changes (makes rollback harder)
- Community reception — Reddit r/openclaw, Discord, GitHub issues for new regressions

As of 2026-05-03, 5.02 has not yet been evaluated or deployed. Still running 4.23.

## Version history on Paul's EC2

| Date       | Version   | Action                          | Reason                            |
|------------|-----------|----------------------------------|-----------------------------------|
| 2026-04-19 | 2026.4.15 | Initial install                  | First setup                       |
| 2026-05-01 | 2026.4.29 | Upgraded via openclaw update     | Latest available                  |
| 2026-05-02 | 2026.4.23 | Downgraded via sudo npm install  | 4.29 too broken (see above)       |

## Model configuration (May 2026)

Primary model changed from openai/gpt-5.4 to anthropic/claude-sonnet-4-6 to work around
the #75824 OpenAI transport bug. gpt-5.4 remains as first fallback.

If/when upgrading to a version that fixes #75824, consider switching primary back to gpt-5.4
and testing with a simple "Hi" before committing.
