# OpenClaw EC2 Monitoring & Telegram Alerting

Set up 2026-06-20 after the gateway OOM death-spiral incident (see
`openclaw-hooks-memory-bug.md` → "Jun 20 2026 — Gateway OOM Death-Spiral").

## Architecture (external watchdog)

The watchdog runs on the **Hermes EC2** (NOT the OpenClaw EC2) so it can still
report when the OpenClaw box is fully down/thrashing. It SSHes in from outside,
checks health, and pings Paul on Telegram only when something is wrong.

```
Hermes cron (*/5) ──► openclaw_watchdog.py ──SSH──► OpenClaw EC2 (gather stats)
                              │
                              └─(problem only)──► hermes send --to telegram ──► Paul's DM
```

## Components

| Piece | Location | Notes |
|-------|----------|-------|
| Watchdog script | `~/.hermes/scripts/openclaw_watchdog.py` (Hermes EC2) | Python, no deps beyond stdlib |
| State file | `~/.hermes/scripts/.openclaw_watchdog_state.json` | tracks last_oom, alerted_sig, last_alert (anti-spam) |
| Cron job | name "OpenClaw EC2 watchdog", id `7fadd28a5351` | `*/5 * * * *`, `no_agent=true`, `deliver=telegram` |
| Delivery | `hermes send --to telegram` | reuses gateway's Telegram creds; home channel chat_id `8634063272` (Paul's DM) |

## Checks performed each run

1. **Reachability** — SSH via private VPC `172.31.20.31`, Tailscale `100.100.35.63` fallback. Neither → "UNREACHABLE" alert.
2. **Gateway** — `systemctl --user is-active openclaw-gateway` == active.
3. **Health** — `curl http://127.0.0.1:18789/health` == 200.
4. **Memory** — available RAM > `LOW_MEM_MB` (1200 MB).
5. **New OOM-kills** — `journalctl -k -b` "Out of memory: Killed" newer than last seen.

## Alert behaviour

- **Silent when healthy** (empty stdout → `no_agent` cron sends nothing).
- New OOM → immediate alert. Persisting condition → repeats at most every `COOLDOWN` (1800 s).
  Changed problem set → re-alerts. One "✅ RECOVERED" message when all checks pass after an alert.

## The Telegram send path (reusable for any alert/script)

`hermes send` reuses the gateway's platform credentials — no token handling needed,
works without the agent loop. The Telegram bot token is in Hermes' secrets store
(NOT in `~/.hermes/.env` or `config.yaml`, so it won't grep out).

```bash
cd /home/ubuntu/.hermes/hermes-agent
./venv/bin/python -m hermes_cli.main send --to telegram "message"        # home channel (Paul DM)
./venv/bin/python -m hermes_cli.main send --list                          # show targets
echo "body" | ./venv/bin/python -m hermes_cli.main send --to telegram --subject "[tag]"
```

`TELEGRAM_HOME_CHANNEL` = `8634063272` (Paul's DM). Target syntax:
`telegram` (home), `telegram:chat_id`, `telegram:chat_id:thread_id`, `telegram:#channel`.

## Managing the job

```
cronjob list            # find it / check last_status
cronjob pause  7fadd28a5351
cronjob resume 7fadd28a5351
cronjob run    7fadd28a5351   # fire once now
cronjob remove 7fadd28a5351
```

## Tuning knobs (edit script top)

- `LOW_MEM_MB` (1200) — available-RAM alert floor.
- `COOLDOWN` (1800) — seconds between repeat alerts for a persisting condition.
- `HOSTS` — SSH targets in priority order.
- Possible additions discussed but not yet added: disk-space check, gateway RSS-vs-cap check.

## Gotcha encountered during setup

The Hermes `write_file`/`patch` tools resolved paths against a Mac prefix
(`/System/Volumes/Data/...`) instead of the remote `/home/ubuntu/...` — the
workdir-injection bug noted in `hermes-to-openclaw-ssh-execution.md`. Workaround:
write the script via `execute_code` + `terminal` using base64 (`echo <b64> | base64 -d > file`),
and edit skill files via `skill_manage` (which targets the skill dir correctly).
