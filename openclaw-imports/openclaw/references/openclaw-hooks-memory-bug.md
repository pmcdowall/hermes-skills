# OpenClaw EC2 Memory / OOM Incidents

> **CURRENT STATUS (2026-06-20):** The hooks bug (#90993) below is FIXED in the
> installed build (2026.6.8) and the reaper workaround has been removed. The
> hooks sections are retained as HISTORICAL reference. The live failure mode as of
> Jun 20 2026 is **gateway memory runaway on an undersized box** — see the section
> immediately below, which is the current operational guidance.

---

## Jun 20 2026 — Gateway OOM Death-Spiral + Fix (CURRENT)

**Symptom (Paul's report):** OpenClaw keeps taking itself offline — gateway restarts
but doesn't come back (or takes ~30 min), and the EC2 has "crashed" several times.

**Diagnosis:**
- Box was **t3a.large (2 vCPU / 8 GB / 2 GB swap)** running OpenClaw **2026.6.8**.
- **6 OOM-kills in ~70 minutes**, victim every time = `openclaw-gateway.service`
  MainThread (Node, ~37 GB *virtual* reservation). It ballooned under load until
  8 GB RAM + 2 GB swap was exhausted, then the whole box thrashed.
- Unreachable-box fingerprint: **TCP port 22 opens but "Connection timed out during
  banner exchange"** + ICMP 100% loss + Tailscale node drops. That = kernel answers
  SYN but userspace is memory-starved. Classic OOM thrash, NOT a network fault.
- **"Restarts but won't come back"** root cause: gateway unit had `Restart=always` +
  `RestartUSec=5s` but `StartLimitBurst=5` / 1-min window. >5 OOM-respawns in a
  minute → systemd gives up and leaves it dead. The ~30-min recovery = box slowly
  reclaiming memory via swap/cache eviction.
- The old hooks-reaper safety net was already gone (correctly removed post-2026.6.6).

**Fixes applied:**
1. **Paul resized the instance → t3a.xlarge (4 vCPU / 16 GB).** This is the real
   root-cause fix — gateway's ~3.4 GB peak is now comfortable headroom. (Resize
   requires a stop/start, so it reboots the box; uptime resets.)
2. **Gateway cgroup guardrail** — separate drop-in (does NOT touch override.conf,
   which holds the API keys):
   `~/.config/systemd/user/openclaw-gateway.service.d/limits.conf`
   ```
   [Unit]
   StartLimitIntervalSec=300
   StartLimitBurst=10
   [Service]
   MemoryHigh=6G
   MemoryMax=8G
   OOMScoreAdjust=600
   ```
   On the 16 GB box: `MemoryMax=8G` kills ONLY the gateway before it suffocates
   SSH + the other ~13 services; `OOMScoreAdjust=600` makes it the kernel's first
   victim; widened StartLimit stops the permanent-dead wedge. (On the old 8 GB box
   a tighter 3G/2.5G cap was used — retune the cap to the box's RAM.)

**Verification after any gateway change:**
```bash
export XDG_RUNTIME_DIR=/run/user/$(id -u)
systemctl --user show openclaw-gateway -p MemoryMax -p OOMScoreAdjust -p ActiveState
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:18789/health   # expect 200
```

**Recovery when the box is mid-thrash and SSH won't answer:** run a background loop
that hammers SSH and lands `sudo pkill -9 -f openclaw-hooks` (or the fix script) the
instant a window opens — recovers in seconds vs waiting ~30 min for the OOM-killer.
No AWS CLI/creds on Hermes EC2 or the Macs (Mac creds = `pandb_r53_user`, Route53-only,
no EC2 perms) — so AWS-side reboot is NOT available; SSH-landing or Paul's console only.

**Monitoring:** an external watchdog now runs on the Hermes EC2 (Hermes cron every 5 min,
delivers to Telegram) and alerts on OOM-kills / gateway-down / low-memory / unreachable.
See `references/external-watchdog-telegram-alerts.md` and `scripts/openclaw_watchdog.py`.

---

## oauth2-proxy secret hardening (Jun 20 2026)

`oauth2-proxy.service` (user unit, `team.onefellswoop.com.au` OIDC) had `--client-secret`
and `--cookie-secret` in plaintext ExecStart args → visible to any local user via `ps`.
Fixed by moving both into `~/.config/oauth2-proxy.env` (mode 600) read via
`EnvironmentFile=`, using env var names `OAUTH2_PROXY_CLIENT_SECRET` /
`OAUTH2_PROXY_COOKIE_SECRET`, and removing the two flag lines. Unit backed up as
`oauth2-proxy.service.bak-<ts>`. Verify: `/ready`→200, unauth `/`→302 to
login.microsoftonline.com, and `ps -eo cmd | grep oauth2-proxy` shows no secrets.

---

# openclaw-hooks Memory Bug — Issue #90993 (HISTORICAL — fixed in 2026.6.6+)

## The Bug
`openclaw-hooks` relay processes (spawned by the gateway for every tool/exec call) never exit.
Each consumes **300–550 MB RSS**. They accumulate with every agent turn. Within minutes the
server OOMs and SSH becomes unreachable, requiring a hard reboot.

**Root cause (two bugs combined in 2026.6.5):**
1. `readStreamText(stdin)` loops with **no timeout** — if the spawning harness holds stdin open,
   the relay blocks forever before any timeout logic applies.
2. After the relay commander action sets `process.exitCode`, there is no `process.exit()`.
   Any still-open handle (gateway WebSocket, stdin) keeps the Node.js process alive indefinitely.

**Affected versions:** 2026.6.1, 2026.6.5  
**Fixed in main:** PR #91550 (merged ~2026-06-09), commit `14b1ebd`  
**Fix ships in:** 2026.6.6 (not yet released as of 2026-06-12)

---

## Reaper Workaround (active on EC2 until 2026.6.6)

Systemd timer that kills hooks processes older than 90 seconds. Runs every 60s.

```
/usr/local/bin/openclaw-hooks-reaper   # kill script
/etc/systemd/system/openclaw-hooks-reaper.service
/etc/systemd/system/openclaw-hooks-reaper.timer
/var/log/openclaw-hooks-reaper.log     # persistent kill log
```

Reaper timeout was tightened from 120s → 90s after a second OOM, then to **45s** after the
Jun 12 2026 event (see post-mortem below). Timer fires every 60s; worst-case accumulation
window is now 105s (45s age threshold + up to 60s until next timer fire).

**Known issue — log file permissions:** The reaper runs as root via systemd but `/var/log/openclaw-hooks-reaper.log`
must be pre-created and owned by root. If the file is missing the `>>` append silently fails
and systemd logs `Failed to start` even though the kills succeed (the kill and log paths are
separate). Fix: `sudo touch /var/log/openclaw-hooks-reaper.log && sudo chmod 644 /var/log/openclaw-hooks-reaper.log`.
The reaper script now falls back to `logger` if the file write fails, so kills are always
recorded in journald even if the flat log is missing.

---

## Dist Patch (active on EC2 — PR #91550 backport)

Patched file: `/usr/lib/node_modules/openclaw/dist/hooks-cli-DTy7ETPt.js`  
Backup: `/usr/lib/node_modules/openclaw/dist/hooks-cli-DTy7ETPt.js.bak-2026.6.5`

The patch translates PR #91550 TypeScript → plain JS and applies:
- `AbortController`-based deadline covering stdin read + bridge + gateway RPC
- `destroyReadableStream()` on deadline fire — actively unblocks the `for await` stdin loop
- Explicit `process.exit()` after relay action completes

**⚠️ Initial patch was incomplete (applied 2026-06-12 00:53):**  
The `callGatewayFn` call was missing `signal: deadline.signal`. This meant the AbortController
fired correctly, but the underlying gateway HTTP/WebSocket request was NOT cancelled — it kept
the Node.js process alive until it completed or timed out independently. The `readStreamText`
abort path was correct; only the gateway fallback path was broken.

**Second patch (applied 2026-06-12 ~01:40):**  
Added `signal: deadline.signal` to `callGatewayFn` call. Now matches PR #91550 source exactly.

**Verified behaviour (2026-06-12, second patch):** relay exits in ~4s (800ms timeout +
gateway negotiation error). Zero lingering processes. The gateway call is now properly
aborted when the deadline fires.

**To apply the patch on a fresh install** (before 2026.6.6):
1. Find the hooks-cli dist file: `grep -rl 'runNativeHookRelayCli' /usr/lib/node_modules/openclaw/dist/`
2. Back it up: `sudo cp <file> <file>.bak`
3. Apply both changes with `sudo python3`:
   - Add `signal: deadline.signal` to the `callGatewayFn({ ... })` call
   - Add `process.exit(process.exitCode ?? 0)` after `process.exitCode = await runNativeHookRelayCli(opts)`
   - Add `AbortController` deadline + `destroyReadableStream` abort listener in `readStreamText`

---

## Upgrade to 2026.6.6 — Full Cleanup Checklist

When 2026.6.6 (or later) ships, PR #91550 is built in. All workarounds must be removed cleanly.
**Do not skip the verification step** — confirm hooks exit before removing the reaper.

### 1. Upgrade normally

Follow the standard `templates/ec2-upgrade-procedure.md`. After upgrade, check:

```bash
openclaw --version
# Expect: 2026.6.6 or later
```

### 2. Verify the fix is present in the new dist file

```bash
# The dist filename may differ in 2026.6.6+ — find it first
NEW_FILE=$(grep -rl 'runNativeHookRelayCli' /usr/lib/node_modules/openclaw/dist/ | head -1)
echo "Dist file: $NEW_FILE"

# Confirm the three native-fix markers are present
grep -c 'AbortController\|destroyReadableStream\|process\.exit' "$NEW_FILE"
# Expect: 3 or more
```

### 3. Smoke-test hooks exit cleanly

```bash
START=$(date +%s%3N)
echo '{}' | timeout 5 node /usr/lib/node_modules/openclaw/dist/hooks-cli-*.js hooks relay \
    --provider test --relay-id test123 --event pre_tool_use --timeout 800 2>&1 || true
END=$(date +%s%3N)
echo "Exited in $((END - START))ms"

sleep 1
ps aux | grep openclaw-hooks | grep -v grep || echo 'PASS: no lingering hooks processes'
```

**Expected:** exits in under 5s, no processes remain.  
**If it hangs: STOP — the fix is not present. Do not remove the reaper.**

### 4. Remove the reaper

```bash
sudo systemctl disable --now openclaw-hooks-reaper.timer
sudo rm /etc/systemd/system/openclaw-hooks-reaper.{service,timer}
sudo rm /usr/local/bin/openclaw-hooks-reaper
sudo rm -f /var/log/openclaw-hooks-reaper.log
sudo systemctl daemon-reload
echo "Reaper removed"
```

Confirm it's gone:
```bash
sudo systemctl status openclaw-hooks-reaper.timer 2>&1 | grep -E 'not-found|No such' && echo 'CONFIRMED GONE'
```

### 5. Remove the dist backup

The upgrade will have overwritten the patched dist file. Remove the stale backup:

```bash
sudo rm -f /usr/lib/node_modules/openclaw/dist/hooks-cli-DTy7ETPt.js.bak-2026.6.5
echo "Backup removed"
```

### 6. Update KNOWN_GOOD_VERSION in the system health check

In `/home/ubuntu/system-check/check.py` around line 210, update:

```python
KNOWN_GOOD_VERSION = '2026.6.6'   # or whatever the actual new version string is
```

Then verify syntax:
```bash
python3 -m py_compile /home/ubuntu/system-check/check.py && echo 'OK'
```

### 7. Update the openclaw skill

- In SKILL.md: update the installed version number
- In `references/version-stability-notes.md`: add 2026.6.6 entry
- In `references/system-health-check.md`: update `KNOWN_GOOD_VERSION`
- This file: remove or archive the "Reaper Workaround" and "Dist Patch" sections (or mark Historical)

---

## Post-Mortem: Jun 12 2026 Memory Spike

**What happened:** OpenClaw was running a long multi-tool-call process. Mid-run it lost
connection and reconnected. The disconnection left hooks relay processes stranded — the dist
patch's AbortController fires on the gateway connection drop, but there was a brief window
during reconnection where new hooks spawned before the old ones were reaped.

**Timeline (UTC):**
- `00:20` — OpenClaw restarted (reboot from earlier OOM)
- `00:22` — Reaper timer restarted after reboot
- `00:53` — Dist patch applied to hooks-cli-DTy7ETPt.js
- `00:55` — Gateway connection failures (`connect_to 127.0.0.1:18789 failed`) — gateway briefly down during reconnect
- `01:29–01:32` — Reaper fired 4 times, killing **29 hooks processes total** (12+8+3+6)
- `01:29` — Reaper log file missing → `Permission denied` errors in systemd journal (kills succeeded, log failed)
- After `01:32` — system stabilised; 878MB swap used, 6.4GB RAM available

**Key finding — the initial dist patch was incomplete.** The `readStreamText` abort path was
correct, but the gateway fallback `callGatewayFn` call was missing `signal: deadline.signal`.
When the deadline fired, the `AbortController` aborted, `withDL` rejected — but the underlying
gateway HTTP/WebSocket request kept running, holding the Node.js process alive. A second patch
(~01:40 UTC) added the missing `signal` field. This is the critical line in PR #91550 that
actually cancels the in-flight network request on timeout.

**Fixes applied (Jun 12 2026):**
1. **Second dist patch:** `signal: deadline.signal` added to `callGatewayFn` call — now matches PR #91550 source exactly
2. Reaper kill threshold tightened: 90s → **45s**
3. Reaper log file created with correct permissions: `root:root 644`
4. Reaper script updated to fall back to `logger` if flat-file write fails

**Remaining risk:** The dist patch reduces but doesn't eliminate accumulation under
disconnect/reconnect with a high tool-call workload. The reaper remains essential until 2026.6.6.
If running another heavy multi-tool process, monitor with `watch -n10 'ps aux | grep openclaw-hooks | grep -v grep'`
from the OpenClaw EC2, or check swap with `free -m`.

---

## Diagnosis Pattern

Signs the hooks bug has triggered:
- `ps aux | grep openclaw-hooks` shows multiple processes with high RSS (400–550MB each)
- `systemd-journald: Under memory pressure, flushing caches` repeating in journal
- SSH becomes slow then unresponsive
- Server requires reboot

**Check reaper log to see if it fired:**
```bash
cat /var/log/openclaw-hooks-reaper.log
journalctl -t openclaw-hooks-reaper --no-pager -n 20
```

**Manual cleanup (if reaper hasn't fired yet):**
```bash
# From Hermes EC2, not from within OpenClaw agent session (spawns more hooks!)
ssh -i ~/.ssh/openclaw-ec2.pem ubuntu@172.31.20.31 \
  "pkill -9 -f 'openclaw-hooks' && echo killed"
```
