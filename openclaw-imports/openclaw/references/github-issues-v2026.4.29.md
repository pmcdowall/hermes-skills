# OpenClaw v2026.4.29 — Known GitHub Issues (as of 2026-05-02)

Source: https://github.com/openclaw/openclaw/issues

## #75844 [open] TUI Fatal Deadlock — "connecting | idle" freeze
Labels: bug, regression
URL: https://github.com/openclaw/openclaw/issues/75844

TUI renders "connecting | idle", gateway logs show code=1008 "connect challenge timeout".
The TUI's JS event loop blocks before it can send the auth frame back to the gateway.
Unresponsive to Ctrl-C. Gateway itself is healthy. Confirmed regression in 2026.4.29.

Environment confirmed affected: WSL2 Ubuntu 24.04, Linux loopback bind, also reproduced
on Paul's EC2 (Ubuntu, loopback, 2026.4.29 a448042).

Workaround: Use web UI at http://127.0.0.1:18789/ (or via Caddy proxy). Gateway and all
channels continue working. TUI is the only broken surface.

Status: Open. No fix merged as of 2026-05-02. clawsweeper[bot] notes current main has
"mitigations for slow Gateway handshakes" but not confirmed to fix WSL2/Linux TUI freeze.


## #75818 [open] Bundled channel extensions fail module resolution on first run
URL: https://github.com/openclaw/openclaw/issues/75818

Bundled channel extensions in dist/extensions/<plugin>/ cannot resolve their own runtime
dependencies via Node module resolution on first run. Errors observed:
  Cannot find module 'grammy'  (telegram)
  Cannot find module '../dist/babel.cjs'  (telegram)

openclaw doctor --fix resolves it by staging plugin runtime deps where they can be found.

Affected channels confirmed: telegram. Likely same bug in discord, nostr, slack.

CONFIRMED ON PAUL'S EC2 (2026-05-02):
  openclaw doctor output: "[channels] failed to load bundled channel telegram: Cannot find module 'grammy'"
  After running openclaw doctor --fix: grammy staged + installed in 78ms. Telegram loaded cleanly
  on next gateway restart — no "Cannot find module" error.

Fix (confirmed working):
  openclaw doctor --fix
  systemctl --user restart openclaw-gateway

Status: Open. Workaround (doctor --fix) fully resolves the issue on this EC2.


## #75868 [open] Telegram markdown messages >4096 chars rejected (regression of #67396)
URL: https://github.com/openclaw/openclaw/issues/75868

sendMessageTelegram only invokes the chunk planner when textMode === "html". The default
"markdown" path sends the entire payload as a single sendMessage call. Any agent response
> 4096 chars fails with: 400: Bad Request: message is too long

Reproduced on 2026.4.29. Regression of #67396 (previously fixed for html mode only).

Impact on Paul's setup: affects any long response sent via Telegram (ariel or default channels).
Workaround: set textMode to "html" in Telegram channel config, or keep responses short.

Status: Open.


## #75852 [open] Telegram polling self-conflicts on macOS 2026.4.29
URL: https://github.com/openclaw/openclaw/issues/75852

On macOS with 2026.4.29, the Telegram provider enters repeated "409 Conflict:
terminated by other getUpdates request" loops even with only one local gateway process.
Stops when gateway is stopped, resumes when restarted — internal lifecycle overlap.

Possibly duplicate of #49822. Adds confirmed macOS reproduction.

Impact on Paul's setup: macOS-specific, so does NOT affect the EC2 gateway directly.
Relevant if a Mac-side gateway is ever run again.

Status: Open.


## #75834 [open] Telegram getMe health checks bypass proxy / 45-53s agent stream latency — BLOCKS ENTIRE GATEWAY on 2026.4.29
URL: https://github.com/openclaw/openclaw/issues/75834

Two issues in one report:
1. Telegram getMe health checks fire fetch-timeout even when a SOCKS5 proxy is configured
   and direct curl --socks5-hostname is fast. Suggests health checks bypass proxy config.
2. Embedded/pi agent runs spend 45-53s before stream-ready on local OpenAI-compatible backend.

CONFIRMED CRITICAL ON PAUL'S EC2 (2026-05-02):
Each getMe health check hangs for 13-56 seconds. With two Telegram bot accounts ([ariel] and
[default]), these checks fire every ~60s and together completely starve the gateway event loop.
Effect: TUI can't connect, web UI chat gets no response (3+ minutes wait, bubble disappears),
models.list takes 194 seconds, sessions.list takes 11+ seconds.

eventLoopDelayP99Ms=56707 — 56 second event loop block. eventLoopUtilization=1.

IMMEDIATE FIX: Disable Telegram channel entirely in openclaw.json, restart gateway.
Gateway startup drops from 24s to 17s. No fetch-timeout noise. Event loop fully clean.
Re-enable when #75834 is patched.

Distinguishing this from zombie-TUI starvation:
  zombie-TUI: many WS "handshake timeout" lines, high CPU in ps aux for openclaw-tui processes
  Telegram fetch-timeout: repeated "[fetch-timeout] fetch timeout reached" for api.telegram.org,
    no zombie processes in ps aux, gateway CPU is normal outside the fetch windows

Status: Open.


## #75656 [open] Synchronous session transcript reads block Gateway event loop
URL: https://github.com/openclaw/openclaw/issues/75656

Root cause: session-utils.fs uses synchronous fs.readFileSync() during agent prep.
With accumulated session transcripts, blocks Node.js event loop for 30-125 seconds.
Manifests as WS handshake timeouts and Telegram unresponsiveness.

Evidence from issue (Raspberry Pi arm64, Node 24.14.1, 2026.4.29):
  core-plugin-tools: 40499ms
  system-prompt:     31392ms
  stream-setup:      32088ms
  totalMs:           125276ms  (125 seconds before any model call)

Also confirmed on Ubuntu x86-64 with 60 active sessions (2026.4.26 and 2026.4.29).
Also confirmed on macOS arm64: sessions.list calls 16-77 seconds each, stacking.

Mitigation: Keep sessions.json trimmed. Disconnect Mac app while troubleshooting TUI
(app's sessions.list polling compounds the blocking).

Status: Open. clawsweeper[bot] notes "partial mitigation for common session-history and
session-list paths" in current main but "synchronous full-transcript reads on Gateway/session
event paths" not fully fixed.


## #75539 [open] Telegram/QQBot fail on servers without global IPv6 (undici IPv6-first)
URL: https://github.com/openclaw/openclaw/issues/75539

Root cause: plugin-runtime-deps bundles undici@8.1.0 which defaults to happy-eyeballs
IPv6-first. On servers where IPv6 resolves but has no global route, each outgoing request
to api.telegram.org waits for IPv6 timeout before falling back to IPv4. Blocks event loop
31+ seconds per cycle during startup.

Symptoms on Paul's EC2 (confirmed):
  [fetch-timeout] fetch timeout reached; aborting operation
  [telegram] fetch fallback: enabling sticky IPv4-only dispatcher (codes=ETIMEDOUT,ENETUNREACH)
  [diagnostic] liveness warning: eventLoopDelayMaxMs=12624.9

Workaround 1 (partial): NODE_OPTIONS=--dns-result-order=ipv4first in systemd override.
  LIMITATION: bundled undici ignores Node.js DNS settings, uses its own resolver.
  The gateway's auto IPv4 fallback ("sticky IPv4-only dispatcher") is the actual mitigant.

Workaround 2: Downgrade to v2026.4.27 (pre-undici@8.1.0).

Status: Open. clawsweeper[bot] notes Telegram has IPv4/fallback handling in current main
but QQBot REST/WebSocket paths still unresolved. Core issue not closed.


## #75759 [open] v2026.4.29: idle gateway burns 14% CPU, Telegram timeouts, event loop delays
URL: https://github.com/openclaw/openclaw/issues/75759

Main thread burns 14% CPU 24/7 at idle (zero active agent runs) after upgrading to 2026.4.29.
Event loop delays 7-9 seconds. Telegram ETIMEDOUT. Downgrade to 2026.4.15 eliminates it.
Likely related to #75539 (undici) and possibly the sync-readFileSync pattern in #75656.

Status: Open. Needs maintainer-led profiling. Not safe for autonomous fix.


## Other relevant open issues

#75289 [open] TUI processes peg 100% CPU and survive terminal close
  openclaw-tui processes spin at 100% CPU per process and don't exit when terminal closes.
  Confirmed on 16GB Mac mini. Matches Paul's zombie process pattern.
  URL: https://github.com/openclaw/openclaw/issues/75289

#75137 [open] TUI process consumes 89-99% CPU at idle (busy-loop)
  openclaw-tui sits at 89-99% CPU even when idle/connected normally.
  URL: https://github.com/openclaw/openclaw/issues/75137

#73323 [open] Gateway runtime degradation: pricing fetch 60s timeouts, Telegram polling stalls
  Chronic across 4.23/4.25/4.26 on Windows 11 + Node 24. Multi-subsystem network degradation.
  URL: https://github.com/openclaw/openclaw/issues/73323

#68944 [open] CLI commands hang at WebSocket gateway handshake
  Every CLI subcommand requiring live gateway connection hangs indefinitely.
  CLI connects at TCP level but blocks before auth. Older issue, still open.
  URL: https://github.com/openclaw/openclaw/issues/68944


## How to search for new issues

GitHub API (no Chrome needed from EC2):
  # Full issue list (50 per page):
  curl -s "https://api.github.com/repos/openclaw/openclaw/issues?state=open&per_page=50" \
    -H "Accept: application/vnd.github+json" > /tmp/oc_issues.json && \
  python3 -c "
  import json
  with open('/tmp/oc_issues.json') as f:
      issues = json.load(f)
  for i in issues:
      labels = '|'.join(l['name'] for l in i['labels'])
      print(f'#{i[\"number\"]}: {i[\"title\"]}  [{labels}]')
  print(f'\nTotal: {len(issues)}')
  "

  # Fetch individual issue body:
  curl -s "https://api.github.com/repos/openclaw/openclaw/issues/<NUMBER>" \
    -H "Accept: application/vnd.github+json" | python3 -c \
    "import json,sys; i=json.load(sys.stdin); print(i['title']); print((i.get('body') or '')[:1000])"

Useful search terms: tui handshake, connect challenge timeout, telegram ETIMEDOUT,
  2026.4.29, event loop delay, synchronous read, sessions.list
