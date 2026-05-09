# OpenClaw 2026.4.29 — Key Changes (Upgrade Review Notes)

Reviewed May 2026 when upgrading from 2026.4.21 on Paul's EC2.

## Breaking Changes

### tools.exec / tools.fs no longer implicitly widen restrictive profiles
- **Affects**: users with tools.profile = "messaging" or "minimal" who also configure tools.exec or tools.fs
- **Does NOT affect**: default "full" profile (Paul's EC2 uses full profile — no impact)
- **Fix if needed**: add explicit `alsoAllow` entries in tools config
- **Detection**: startup warning emitted if affected config is found

## Notable Fixes Relevant to This Install

### Subagent orphan recovery (Fixes #74864)
Previously required manual sessions.json surgery to remove dead subagent entries.
2026.4.29 adds automatic orphan recovery with persisted recovery attempts and
wedged-session tombstones. doctor/maintenance now reconciles these sessions.

### Plugin runtime-deps staging fixes (multiple PRs)
- Stale symlinked mirror roots no longer crash on update
- Staged package entry verification before reuse (fixes incomplete ajv/MCP SDK installs)
- pnpm install no longer relaunches after gateway startup (reduces event-loop pressure)
This explains the dramatic startup improvement (135-145s → ~5s) observed after upgrade.

### Gateway/systemd exit code 78 (Fixes #75115)
Gateway now exits with sysexits 78 for supervised lock and EADDRINUSE conflicts.
`RestartPreventExitStatus=78` stops `Restart=always` loops instead of repeatedly
reloading plugins against an occupied port. Better than the stop+start workaround.

### Cron: validate disabled job schedule edits (Fixes #74459)
Invalid cron changes no longer partially mutate stored jobs.

### Session abort alignment
Abort RPCs now return after targeted sessions actually halt (not resolving early).

## Changes Not Relevant to This Install (but worth knowing)

- NVIDIA provider added
- Active Memory per-conversation allowedChatIds/deniedChatIds filters
- Memory people wiki (canonical aliases, person cards, relationship graphs)
- Bedrock Opus 4.7 full thinking profile (xhigh, adaptive, max)
- Many Telegram/Discord/Slack/Feishu channel fixes
- New commitments system (opt-in follow-up reminders via heartbeat)
- OpenGrep security scanning added to CI

## What Was in Main Branch (Unreleased) at Time of Review

Runtime-deps repair fix, TTS provider fallback fix, session abort alignment, agent output
fix (Telegram assistant metadata leak), plugin lazy activation fix. These will land
in the next release — not critical for current install.
