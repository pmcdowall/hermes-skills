---
name: openclaw
description: Troubleshoot and operate OpenClaw — a self-hosted AI agent gateway platform similar to Hermes. Covers installation, configuration, gateway management, channels, tools, providers, and common failures.
triggers:
  - openclaw is not working
  - openclaw gateway
  - openclaw install
  - openclaw troubleshoot
  - openclaw config
  - openclaw channel
  - openclaw doctor
confirmed: true
---

# OpenClaw — Operations & Troubleshooting

## What is OpenClaw?

OpenClaw (https://openclaw.ai) is a self-hosted, MIT-licensed AI agent gateway platform.
Conceptually identical to Hermes: connects messaging platforms (WhatsApp, Telegram, Discord,
Slack, Signal, iMessage, etc.) to AI model providers (Anthropic, OpenAI, Gemini, local via
Ollama, etc.) and gives a 24/7 AI assistant that can take real-world actions.

- GitHub: https://github.com/openclaw/openclaw (~360k stars)
- Docs: https://docs.openclaw.ai
- Skills marketplace: https://clawhub.ai
- Discord: https://discord.gg/openclaw
- Security: https://trust.openclaw.ai
- Creator: Peter Steinberger (@steipete, now at OpenAI)
- Runtime: Node.js (TypeScript). Node 24 recommended, Node 22.14+ minimum.
- Config: ~/.openclaw/openclaw.json
- Default port: 18789 (localhost)


## Architecture

Single long-lived Gateway daemon is the core:

  Chat Apps (WhatsApp/Telegram/Discord/etc.)
       |
  [Gateway daemon — port 18789]
       |
  AI Model Providers (Anthropic/OpenAI/Gemini/Ollama/etc.)

- One Gateway per host
- Clients (macOS app, CLI, web UI) connect via WebSocket
- Nodes (iOS/Android/headless) connect as role:node
- WebChat served at http://127.0.0.1:18789/
- OpenAI-compatible API at /v1/chat/completions, /v1/models, /v1/embeddings


## Installation

Recommended (macOS/Linux/WSL2): curl the install script from openclaw.ai/install.sh
npm: npm install -g openclaw@latest, then openclaw onboard --install-daemon
From source: clone github.com/openclaw/openclaw, pnpm install && pnpm build, then link globally

Verify install:
  openclaw --version
  openclaw doctor
  openclaw gateway status

Service management (post-install):
- macOS: LaunchAgent (ai.openclaw.gateway)
- Linux/WSL2: systemd user service (openclaw-gateway.service)
- Windows: Scheduled Task


## Diagnosing a Stuck Gateway from Outside (EC2/VPS)

When the gateway is unresponsive, the OpenClaw agent cannot fix itself — any exec tool call
hangs waiting for a response that never comes, and the agent kills its own session.
The correct approach is to connect via external SSH and diagnose/fix from there.

On the Hermes/Rocky side (or any external machine with SSH access):
  ssh -i /path/to/key.pem ubuntu@<ec2-ip>

Immediate triage commands once connected:
  ps aux | grep openclaw | grep -v grep        # how many processes, CPU%, RAM%
  free -h                                      # is OOM killer a factor?
  ls -lah ~/.openclaw/flows/ ~/.openclaw/tasks/ # WAL file sizes
  ls -lah ~/.openclaw/agents/main/sessions/sessions.json  # sessions.json size
  journalctl --user -u openclaw-gateway --since "10 minutes ago" --no-pager | tail -40

Key things to look for in logs:
  "[session-write-lock] releasing lock held for NNNNNms" — sessions.json bloat (see fix below)
  "EADDRINUSE" — port conflict / zombie process
  "SIGUSR1" restart loop — config reload bug (v2026.4.15 Linux/systemd)
  WAL files >1MB — SQLite checkpoint stall

Safe process cleanup (kill children, not the gateway):
  # Identify gateway PID (the one running as 'openclaw-gateway')
  # Kill only child openclaw processes (orphaned CLI, TUI, doctor instances)
  # NEVER kill -9 all openclaw processes blindly — check PIDs first

To stop the gateway cleanly for maintenance without auto-restart:
  systemctl --user stop openclaw-gateway
  systemctl --user disable openclaw-gateway   # prevents auto-restart
  pkill -9 -f openclaw                        # clean up stragglers
  # ... do your fixes ...
  systemctl --user enable openclaw-gateway
  systemctl --user start openclaw-gateway


## First 60 Seconds of Troubleshooting

BEFORE checking gateway or auth, do these three checks first:

  1. Check for zombie TUI processes on BOTH EC2 AND MAC:
     ps aux | grep openclaw | grep -v grep
     Look for: openclaw-tui processes (not openclaw-gateway). Any at 70%+ CPU are zombies.
     These starve the gateway event loop and cause handshake timeouts.
     Fix (EC2): pkill -9 -f openclaw-tui
     Fix (Mac): kill -9 <pid> for each zombie (use specific PIDs)
     IMPORTANT: After killing zombies on EC2, check `which node` — new SSH sessions
     are login shells and may have linuxbrew node v25.9.0 in PATH. Always source
     ~/.bashrc or verify `which node` shows v24.15.0 before retrying TUI.
     Fix EC2: pkill -9 -f openclaw-tui
     Fix Mac: kill -9 <specific pids> (check ps output first)

  2. Check gateway.remote.url on Mac (if Mac TUI is hanging):
     cat ~/.openclaw/openclaw.json | python3 -c "import json,sys; c=json.load(sys.stdin); print(c.get('gateway',{}).get('remote',{}))"
     Must be: wss://rocky.onefellswoop.com.au/openclaw
     If ws://localhost:18789 or similar — that is the hang. Fix immediately:
     openclaw config set gateway.remote.url wss://rocky.onefellswoop.com.au/openclaw
     openclaw gateway restart

  3. Check if Mac TUI is even reaching the EC2 gateway:
     Watch EC2 logs while Mac TUI connects:
     journalctl --user -u openclaw-gateway --no-pager -n 10
     If NO new entries appear when Mac TUI starts — the Mac is not reaching EC2 at all.
     (Node host connecting shows res ok bursts; TUI not reaching = URL problem on Mac)

  4. After a reboot or gateway restart, wait ~60 seconds before attempting TUI connections.
     The Telegram plugin's undici IPv6-first startup (v2026.4.29 issue #75539) blocks the event
     loop for up to 31 seconds until the gateway triggers its IPv4-only fallback. Watch logs for:
       "enabling sticky IPv4-only dispatcher"
     Once that line appears, the startup event loop spike is over and TUI attempts are safer.

Then run this ladder in order:
  openclaw status
  openclaw status --all
  openclaw gateway probe
  openclaw gateway status
  openclaw doctor
  openclaw channels status --probe
  openclaw logs --follow

Good signs:
- openclaw status — configured channels, no auth errors
- openclaw gateway status — "Runtime: running" and "RPC probe: ok"
- openclaw gateway probe — "Reachable: yes"
- openclaw doctor — no blocking errors
- openclaw channels status --probe — "works" or "audit ok" per channel
- openclaw logs --follow — steady activity, no repeating fatal errors


## EC2/VPS — Known Gateway Instability Causes (Discovered in Production)

These were found on a real Ubuntu EC2 deployment running 2026.4.15. All contributed to
the gateway appearing "alive" (port 18789 listening) but completely unresponsive to CLI
commands — the classic stuck event loop symptom.

### 1. SQLite WAL file bloat (primary cause of memory leak and event loop stalls)

OpenClaw uses SQLite for flows and task runs. The WAL (write-ahead log) files can grow
unbounded if checkpointing stalls. A 4MB WAL on an 84KB database forces the gateway to
replay the entire log on every read, progressively eating memory and CPU.

Symptom: gateway accepts connections but CLI commands time out. High CPU (60-100%).
Memory grows over hours. `ps aux` shows openclaw-gateway at 500MB+.

Diagnosis:
  ls -lah ~/.openclaw/flows/registry.sqlite*
  ls -lah ~/.openclaw/tasks/runs.sqlite*
  (WAL file much larger than the main .sqlite file = problem)

Fix (must stop gateway first):
  systemctl --user stop openclaw-gateway
  systemctl --user disable openclaw-gateway   # prevent auto-restart during fix
  pkill -9 -f openclaw                         # kill any stragglers
  # install sqlite3 if needed: sudo apt-get install -y sqlite3
  sqlite3 ~/.openclaw/flows/registry.sqlite "PRAGMA integrity_check; PRAGMA wal_checkpoint(TRUNCATE);"
  sqlite3 ~/.openclaw/tasks/runs.sqlite "PRAGMA integrity_check; PRAGMA wal_checkpoint(TRUNCATE);"
  sqlite3 ~/.openclaw/memory/main.sqlite "PRAGMA integrity_check;"
  systemctl --user enable openclaw-gateway
  systemctl --user start openclaw-gateway

After checkpoint: WAL files disappear, main .sqlite file grows to absorb their content.

### 2. Amazon Bedrock plugin model discovery (causes 117-second startup delay)

If the amazon-bedrock plugin is installed with discovery.enabled=true, it calls AWS APIs
on every gateway startup to enumerate available models. On a VPS without fast AWS network
access this blocks startup for 60-120+ seconds, during which the gateway is unresponsive.

Symptom: journalctl shows 2+ minutes between "starting..." and "ready".

Fix: disable discovery in ~/.openclaw/openclaw.json:
  "plugins": {
    "entries": {
      "amazon-bedrock": {
        "config": {
          "discovery": { "enabled": false }
        },
        "enabled": true
      }
    }
  }

### Stale plugin registry blocking auth resolution / causing slow agent prep (2026.4.29)

Symptom: `openclaw doctor` reports "Persisted plugin registry is missing or stale."
  TUI connects at TCP level but gateway logs show repeated "handshake timeout" — the
  gateway blocks at "resolving authentication" before completing the WebSocket handshake.
  Running `openclaw tui --help` times out — the CLI binary itself hangs on init.

Also: gateway logs show agent prep taking 50+ seconds:
  "core-plugin-tools:23717ms ... system-prompt:12715ms ... totalMs=54959ms"
  This is the same root cause — stale plugin registry forces full re-resolution on every
  agent turn, starving the event loop so TUI handshakes time out. Even if the TUI symptom
  looks different (Mac: hangs at "connecting | idle"; EC2: "gateway closed (1000)"), check
  the gateway logs for slow prep times as a secondary confirmation.

Diagnosis confirmation via journalctl:
  journalctl --user -u openclaw-gateway --no-pager -f
  Watch for: "[agent/embedded] [trace:embedded-run] prep stages: ... totalMs=NNNNNms"
  If totalMs > 20000ms (20 seconds), stale plugin registry is likely the cause.

Cause: The plugin registry (plugins/installs.json) is missing or out of date. In 2026.4.29,
  the gateway's auth resolution phase reads the plugin registry. A stale registry causes
  auth to block indefinitely, timing out every incoming TUI connection.

Fix:
  openclaw doctor --fix
  # Look for: "Plugin registry rebuilt: 51/52 enabled plugins indexed."
  # Then restart the gateway:
  systemctl --user restart openclaw-gateway

Note: `openclaw doctor --fix` also updates the systemd service file (backs up old one to
  openclaw-gateway.service.bak). After running it, verify ExecStart is still correct:
  grep ExecStart ~/.config/systemd/user/openclaw-gateway.service
  Then: systemctl --user daemon-reload

Note: doctor itself may time out at the end (last phase hangs for 60+ seconds on this EC2)
  but that's OK — the important actions (registry rebuild, service update) complete early.

### 3. Runaway openclaw-doctor / orphaned CLI processes

`openclaw doctor` spawned from a TUI session can run indefinitely at 70-110% CPU if the
gateway becomes unresponsive (it waits for responses that never come). These accumulate
and together can consume all available CPU and RAM, triggering the OOM killer which then
kills npm installs and other processes.

Symptom: `ps aux | grep openclaw` shows openclaw-doctor at high CPU, multiple openclaw
processes, and npm installs being SIGKILL'd mid-download.

Fix: Kill by PID — do NOT use `killall openclaw` as it will also kill the gateway.
  ps aux | grep openclaw   # identify PIDs
  kill -9 <pid-of-doctor> <pid-of-orphaned-tui-processes>
  # leave the main openclaw-gateway PID alone if you want to keep it running

### Service entrypoint mismatch after reinstall / update

After an npm reinstall or update, the systemd service file may still point to the old
binary path (e.g. /home/linuxbrew/.linuxbrew/lib/node_modules/openclaw/dist/index.js)
while the current install is at /usr/lib/node_modules/openclaw/dist/index.js.
This causes systemd to silently launch the wrong (old) version.

Symptom: `openclaw doctor` reports "Gateway service entrypoint does not match current install."
Also: version mismatches between `openclaw --version` and gateway logs.

Fix: Run `openclaw doctor` — it will offer to update the service file automatically.
Or manually update ExecStart in:
  ~/.config/systemd/user/openclaw-gateway.service
Then: systemctl --user daemon-reload && systemctl --user restart openclaw-gateway

IMPORTANT (confirmed May 2026): After sudo npm install -g openclaw@<version>, ALWAYS check
the ExecStart line in the service file:
  grep ExecStart ~/.config/systemd/user/openclaw-gateway.service
Correct form on this EC2:
  ExecStart=/usr/bin/node /usr/lib/node_modules/openclaw/dist/index.js gateway --port 18789
If it still shows linuxbrew paths, patch it with sed:
  sed -i "s|ExecStart=.*|ExecStart=/usr/bin/node /usr/lib/node_modules/openclaw/dist/index.js gateway --port 18789|" \
    ~/.config/systemd/user/openclaw-gateway.service
Then: systemctl --user daemon-reload
  which openclaw  (compare paths)

Manual fix (faster than doctor):
  sed -i "s|ExecStart=.*|ExecStart=/usr/bin/node /usr/lib/node_modules/openclaw/dist/index.js gateway --port 18789|" \
    ~/.config/systemd/user/openclaw-gateway.service
  grep ExecStart ~/.config/systemd/user/openclaw-gateway.service  # verify
  systemctl --user daemon-reload

Or run `openclaw doctor` — it will offer to update the service file automatically.
Then: systemctl --user enable openclaw-gateway && systemctl --user start openclaw-gateway

### 5. Gateway auth token becomes a placeholder

After repeated doctor/restart cycles, gateway.auth.token in openclaw.json can become
corrupted to a placeholder value like "e26068...0034". This causes all clients (Mac app,
TUI, nodes) to get "token_mismatch" rejections every 10 seconds, flooding logs and
wasting CPU.

NOTE: `openclaw config get gateway.auth.token` always shows __OPENCLAW_REDACTED__ — this
is CLI display redaction, not the actual value. Read the real value directly:
  python3 -c "import json; d=json.load(open('/home/ubuntu/.openclaw/openclaw.json')); print(d['gateway']['auth']['token'])"

To regenerate a clean token:
  python3 -c "
  import json, secrets
  path = '/home/ubuntu/.openclaw/openclaw.json'
  d = json.load(open(path))
  tok = secrets.token_hex(16)
  d['gateway']['auth']['token'] = tok
  open(path, 'w').write(json.dumps(d, indent=2))
  print(tok)
  "
Then restart the gateway and update the token in any connected Mac/iOS apps.

### 6. Startup performance optimisations (EC2/low-power hosts)

Add to ~/.config/systemd/user/openclaw-gateway.service.d/override.conf:
  Environment="NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache"
  Environment="OPENCLAW_NO_RESPAWN=1"

And create the cache dir:
  sudo mkdir -p /var/tmp/openclaw-compile-cache
  sudo chmod 777 /var/tmp/openclaw-compile-cache

Then: systemctl --user daemon-reload && systemctl --user restart openclaw-gateway


## Common Failure Signatures

Error: "refusing to bind gateway ... without auth"
  Cause: Non-loopback bind without gateway auth
  Fix: Add gateway.auth.token to config or use --auth flag

Error: "EADDRINUSE" or "another gateway instance already listening"
  Cause: Port conflict
  Fix: Kill existing process or start with --force flag

Error: "Gateway start blocked: set gateway.mode=local"
  Cause: Config set to remote mode or local-mode stamp missing
  Fix: Check gateway.mode in config

Error: "unauthorized during connect"
  Cause: Auth token mismatch between client and gateway
  Fix: openclaw doctor --generate-gateway-token

Error: "HTTP 429: rate_limit_error: Extra usage required for long context"
  Cause: Anthropic long-context rate limit
  Fix: Use /compact, switch model, or see docs.openclaw.ai/gateway/troubleshooting

Error: "messages[].content expecting a string"
  Cause: Local OpenAI-compatible backend compat issue
  Fix: Set models.providers.<provider>.models[].compat.requiresStringContent: true in config

Error: "package.json missing openclaw.extensions"
  Cause: Old-format plugin
  Fix: Add openclaw.extensions pointing to ./dist/index.js in plugin's package.json

Error: "openclaw not found" (after install)
  Cause: npm global bin not in PATH
  Fix: Run `npm prefix -g` to find global bin dir, add it to PATH in ~/.bashrc or ~/.zshrc


## Gateway Operations

Status and health:
  openclaw gateway status
  openclaw gateway status --deep
  openclaw health
  openclaw status --deep --usage

Start / stop / restart:
  openclaw gateway start
  openclaw gateway stop
  openclaw gateway restart
  openclaw gateway --port 18789 --verbose    (run in foreground)
  openclaw gateway --force                    (kill existing listener first)

Install/uninstall service:
  openclaw gateway install
  openclaw gateway uninstall

Logs:
  openclaw logs --follow
  openclaw logs --limit 200
  openclaw logs --plain

Linux systemd logs (EC2/VPS):
  journalctl --user -u openclaw-gateway -f

Remote access (preferred: Tailscale/VPN). SSH tunnel fallback:
  ssh -N -L 18789:127.0.0.1:18789 user@host
  (SSH tunnels do NOT bypass gateway auth)


## Configuration

Config file: ~/.openclaw/openclaw.json

View and edit:
  openclaw config file            (show path)
  openclaw config get <path>      (e.g. openclaw config get agents.defaults.model)
  openclaw config set <path> <value>
  openclaw config validate
  openclaw config schema

Minimal config example:
  { "agents": { "defaults": { "model": { "primary": "anthropic/claude-opus-4-6" } } } }

Tool profiles (set in config as tools.profile):
  full | coding | messaging | minimal

Tool allow/deny example:
  { "tools": { "allow": ["group:fs", "browser", "web_search"], "deny": ["exec"] } }

Sandbox for group chats:
  { "agents": { "defaults": { "sandbox": { "mode": "non-main" } } } }

Filesystem hardening (recommended):
  { "tools": { "exec": { "applyPatch": { "workspaceOnly": true } }, "fs": { "workspaceOnly": true } } }


## Channels

Supported: WhatsApp, Telegram, Discord, Slack, Signal, BlueBubbles (iMessage),
Google Chat, Microsoft Teams, Matrix, IRC, Feishu, LINE, Mattermost, Nextcloud Talk,
Nostr, QQ Bot, Synology Chat, Tlon, Twitch, Zalo, Zalo Personal, WeChat, WebChat.

Commands:
  openclaw channels list
  openclaw channels status
  openclaw channels status --probe
  openclaw channels add --channel telegram --account main --token $TOKEN
  openclaw channels add --channel discord --account work --token $TOKEN
  openclaw channels remove --channel discord --account work --delete

Pairing (DM security — default dmPolicy is "pairing"):
  openclaw pairing list
  openclaw pairing approve <channel> <code>

Fastest to set up: Telegram (just a bot token).
WhatsApp: requires QR pairing, stores more state on disk.


## Models / Providers

Commands:
  openclaw models list
  openclaw models list --all
  openclaw models status --probe
  openclaw models set anthropic/claude-opus-4-6
  openclaw models auth add
  openclaw models auth login

Supported providers (key ones):
- anthropic — Claude (API key or Claude Max/Pro subscription)
- openai — GPT-4, GPT-5, o1, Codex
- google — Gemini 2.5 Pro/Flash
- ollama — Local models (no API key needed)
- lmstudio — Local models
- openrouter — Unified gateway (100s of models)
- deepseek — DeepSeek V3/R1
- groq — Fast inference
- perplexity — Search-augmented AI
- xai — Grok 3/4

Full provider list: https://docs.openclaw.ai/providers


## Skills and Plugins

Skills (teach agent how to use tools):
  openclaw skills list
  openclaw skills search <query>
  openclaw skills install <name>
  openclaw skills update
  Browse: https://clawhub.ai

Plugins:
  openclaw plugins list
  openclaw plugins install <package>
  openclaw plugins update
  openclaw plugins enable/disable <name>
  openclaw plugins doctor


## Security

  openclaw security audit
  openclaw security audit --deep
  openclaw security audit --fix
  openclaw secrets audit
  openclaw secrets reload
  Security contact: security@openclaw.ai


## Memory

  openclaw memory status
  openclaw memory status --deep --fix
  openclaw memory index
  openclaw memory search "<query>"


## Cron / Scheduled Jobs

  openclaw cron list
  openclaw cron status
  openclaw cron add --name "daily-brief" --every 24h --message "Good morning briefing"
  openclaw cron edit <id>
  openclaw cron rm <id>
  openclaw cron enable/disable <id>
  openclaw cron runs --id <id>
  openclaw cron run <id>    (trigger immediately)


## Agents

  openclaw agents list
  openclaw agents add <name> --workspace <dir> --model <id>
  openclaw agents bind --agent <id> --channel telegram
  openclaw agents delete <id>

Run a single agent turn:
  openclaw agent --message "Ship checklist" --thinking high
  openclaw agent -m "Fix failing tests" --local


## In-Chat Commands

/status, /new, /reset, /compact, /think <level>, /verbose, /usage, /restart, /activation
Thinking levels: off | minimal | low | medium | high | xhigh


## Browser Automation

  openclaw browser status / start / stop
  openclaw browser tabs / open <url> / focus <id> / close <id>
  openclaw browser screenshot / snapshot / navigate <url>
  openclaw browser click <ref> / type <ref> <text> / press <key>
  openclaw browser evaluate --fn <code>


## Upgrading OpenClaw (Safe Procedure for EC2/VPS)

Upgrading via `openclaw update` or `npm install -g openclaw@<version>` on a running gateway
can cause OOM kills mid-install (npm needs ~1.5GB for native compile) and service entrypoint
mismatches. Always follow this sequence:

  1. Stop gateway cleanly (do NOT use restart — see systemctl restart pitfall above):
       systemctl --user stop openclaw-gateway
       systemctl --user disable openclaw-gateway
       pkill -9 -f openclaw

  2. Check sessions.json size and trim if needed before upgrading (pre-upgrade is the right time):
       ls -lah ~/.openclaw/agents/main/sessions/sessions.json
       python3 -c "import json; d=json.load(open('/home/ubuntu/.openclaw/agents/main/sessions/sessions.json')); print(len(d), 'entries')"

  3. Run the install (gateway is stopped, RAM is free):
       npm install -g openclaw@<version>
       # or: openclaw update --channel stable

  4. Run doctor to check for service entrypoint mismatch (common after npm reinstall):
       openclaw doctor
       # If it reports entrypoint mismatch, let it fix — or manually update ExecStart in:
       # ~/.config/systemd/user/openclaw-gateway.service
       # Then: systemctl --user daemon-reload

  5. Re-enable and start (use start NOT restart — avoids SIGTERM-mid-boot):
       systemctl --user enable openclaw-gateway
       systemctl --user start openclaw-gateway

  6. Wait 2-3 minutes (startup is slow on t-class EC2), then verify:
       ss -tlnp | grep 18789
       timeout 30 openclaw gateway probe

### Reading GitHub Issues (no browser needed)

Chrome/browser is not available on EC2. Use the GitHub API directly:

  # List all open issues (50 per page):
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

  # Fetch a specific issue body:
  curl -s "https://api.github.com/repos/openclaw/openclaw/issues/75818" \
    -H "Accept: application/vnd.github+json" | python3 -c \
    "import json,sys; i=json.load(sys.stdin); print(i['title']); print((i.get('body') or '')[:1000])"

Save to file first, then parse — avoids the curl|python3 pipe security flag.
Known issues are documented in: references/github-issues-v2026.4.29.md


## How to review a release before upgrading

Fetch the changelog and extract the target version section:
  curl -fsSL https://raw.githubusercontent.com/openclaw/openclaw/main/CHANGELOG.md | \
    awk '/^## <TARGET_VERSION>/,/^## <PRIOR_VERSION>/'

List all version section headers to find the right anchors:
  curl -fsSL https://raw.githubusercontent.com/openclaw/openclaw/main/CHANGELOG.md | \
    grep '^## '

Things to check in a release changelog before upgrading:
- Any "Changes" entries under Security/tools, tools.exec, or tools.fs (profile widening rules changed in 2026.4.29)
- Any gateway/systemd changes (exit codes, restart behavior)
- Plugin runtime-deps changes (these affect your 6 active plugins)
- sessions.json / orphan recovery changes (may affect how you pre-trim)
- Node.js engine requirement (check `engines.node` in package.json on the tag)

### Breaking change: tools.exec no longer widens restrictive profiles (2026.4.29+)

From 2026.4.29, configuring `tools.exec` or `tools.fs` sections does NOT automatically
enable those tools when using a restrictive profile (`messaging`, `minimal`). You must
add explicit `alsoAllow` entries.

Only affects you if tools.profile is set to "messaging" or "minimal". If no profile is set
(default "full"), behavior is unchanged. Check your current profile:
  python3 -c "import json; d=json.load(open('/home/ubuntu/.openclaw/openclaw.json')); print(d.get('tools',{}).get('profile','not set (full)'))"

If you use a restrictive profile and need exec/fs, add:
  { "tools": { "profile": "messaging", "alsoAllow": ["exec", "group:fs"] } }

A startup warning will appear if affected configs are detected — check `openclaw logs` after
first boot on 2026.4.29.

### Gateway/systemd exit-78 for lock and EADDRINUSE conflicts (2026.4.29+)

From 2026.4.29, the gateway exits with code 78 (sysexits EX_CONFIG) on supervised lock
conflicts and EADDRINUSE errors. If your systemd service has `RestartPreventExitStatus=78`,
this stops restart loops on port conflicts rather than spinning indefinitely.

Recommended addition to the systemd service override:
  mkdir -p ~/.config/systemd/user/openclaw-gateway.service.d/
  cat >> ~/.config/systemd/user/openclaw-gateway.service.d/override.conf << 'EOF'
  RestartPreventExitStatus=78
  EOF
  systemctl --user daemon-reload

### Automatic orphan/wedged-session recovery (2026.4.29+)

From 2026.4.29, the gateway automatically detects and tombstones wedged subagent sessions,
and `openclaw doctor` reconciles them. This reduces the need for manual sessions.json surgery
to clear dead subagent entries.

However: still trim sessions.json before upgrading if it's grown large. The recovery logic
runs against what's there — don't let it inherit 60+ stale entries as its first job.
The session.maintenance config settings (pruneAfter: "7d", maxEntries: 100) remain the
right long-term prevention.

## Update / Reset / Uninstall

Full EC2 upgrade runbook: templates/ec2-upgrade-procedure.md

Update:
  openclaw update
  openclaw update --channel stable|beta|dev
## Update / Reset / Uninstall

Update:
  openclaw update
  openclaw update --channel stable|beta|dev
  openclaw update status

Reset:
  openclaw reset --scope config
  openclaw reset --scope config+creds+sessions
  openclaw reset --scope full

Uninstall:
  openclaw uninstall --all


## EC2 Update Procedure (sudo npm install, confirmed May 2026)

Use this sequence when updating openclaw on this EC2 via npm (NOT openclaw update):

  1. Pre-flight checks:
       ls -lah ~/.openclaw/agents/main/sessions/sessions.json   # target <200KB, <15 entries
       ls -lah ~/.openclaw/flows/*.sqlite* ~/.openclaw/tasks/*.sqlite*  # check WAL sizes
       ps aux | grep openclaw | grep -v grep   # check for child process pile-up

  2. Stop gateway cleanly (disable FIRST to prevent auto-restart):
       systemctl --user disable openclaw-gateway
       systemctl --user stop openclaw-gateway
       pkill -9 -f openclaw-gateway 2>/dev/null
       # Verify: ps aux | grep openclaw-gateway | grep -v grep  (should be empty)

  3. WAL checkpoint while gateway is stopped:
       sqlite3 ~/.openclaw/tasks/runs.sqlite "PRAGMA wal_checkpoint(TRUNCATE);"
       sqlite3 ~/.openclaw/flows/registry.sqlite "PRAGMA wal_checkpoint(TRUNCATE);"

  4. Install new version (needs sudo):
       sudo npm install -g openclaw@<version>
       # Expect 40-60 seconds. If OOM-killed, gateway was not stopped cleanly — retry.

  5. Fix service entrypoint (ALWAYS do this after npm install):
       grep ExecStart ~/.config/systemd/user/openclaw-gateway.service
       # If it shows old path (linuxbrew), patch it:
       sed -i "s|ExecStart=.*|ExecStart=/usr/bin/node /usr/lib/node_modules/openclaw/dist/index.js gateway --port 18789|" \
         ~/.config/systemd/user/openclaw-gateway.service
       systemctl --user daemon-reload

  6. Re-enable and start:
       systemctl --user enable openclaw-gateway
       systemctl --user start openclaw-gateway

  7. Wait for gateway (startup is ~37s in v2026.4.29 — port should be up in under a minute):
       watch ss -tlnp | grep 18789
       # Or: journalctl --user -u openclaw-gateway -f | grep "ready\|error"

  8. Post-update verification:
       openclaw --version                    # confirm new version
       openclaw gateway status               # runtime: running, connectivity: ok
       openclaw channels status              # Telegram ariel + default: connected

  9. Check for linuxbrew node conflict if TUI hangs (see Known Issues section).



## EC2 / VPS Deployment Notes

- Default bind is loopback. Gateway is NOT exposed to the internet by default.
- Use Tailscale or SSH tunnel for remote access. Do NOT expose port 18789 publicly.
- Install Node 24 first: see https://docs.openclaw.ai/install/node
- Config: ~/.openclaw/openclaw.json
- State dir: ~/.openclaw/ (override with OPENCLAW_STATE_DIR env var)

PATH fix if openclaw not found after npm global install:
  node -v                    (confirm Node installed)
  npm prefix -g              (find global bin dir)
  echo $PATH                 (check if global bin is in PATH)
  Add global bin to PATH in ~/.bashrc or ~/.zshrc, then source it.

Check gateway on EC2:
  openclaw gateway status
  systemctl --user status openclaw-gateway    (Linux systemd)
  openclaw logs --follow


## Known Issues and Pitfalls (from real troubleshooting, April 2026)

### sessions.json bloat causing session write lock timeouts (v2026.4.15)
Symptom: Gateway logs show "[session-write-lock] releasing lock held for 80000-115000ms
  (max=15000ms)". CLI commands time out. Chat.send requests fail with gateway timeout.
  TUI disconnects with "gateway request timeout" or "closed | idle".
Cause: sessions.json grows to 1.5-2MB+ as cron run histories, old TUI sessions, and dead
  subagents accumulate. OpenClaw holds an exclusive write lock on this file during every
  agent turn. A 2MB file takes 80-115 seconds to write — blocking everything else.
Note (observed April 2026): Even 553KB / 17 entries caused slow startup. Trim aggressively.
  After trim to 9 entries / 289KB, startup was healthy. Threshold is lower than initially
  thought — keep sessions.json under ~200KB as a target, not just "not MB-scale".
Diagnosis:
  ls -lah ~/.openclaw/agents/main/sessions/sessions.json
  python3 -c "import json; d=json.load(open('/home/ubuntu/.openclaw/agents/main/sessions/sessions.json')); print(len(d), 'entries')"
  (anything over ~200KB or 15+ entries warrants a trim)
  python3 -c "import json; d=json.load(open('/home/ubuntu/.openclaw/agents/main/sessions/sessions.json')); print(len(d), 'entries')"
Fix — trim dead entries (safe to do while gateway is running):
  cp ~/.openclaw/agents/main/sessions/sessions.json ~/.openclaw/agents/main/sessions/sessions.json.bak
  python3 -c "
  import json, os
  path = '/home/ubuntu/.openclaw/agents/main/sessions/sessions.json'
  d = json.load(open(path))
  keep = {k: v for k, v in d.items() if ':run:' not in k and ':subagent:' not in k and ':tui-' not in k}
  open(path, 'w').write(json.dumps(keep, indent=2))
  print(f'Trimmed {len(d)} -> {len(keep)} entries, {os.path.getsize(path)} bytes')
  "
  # Then clear any stale lock files:
  rm -f ~/.openclaw/agents/main/sessions/*.lock
Prevention — tighten session maintenance in openclaw.json:
  "session": {
    "maintenance": {
      "mode": "enforce",
      "pruneAfter": "7d",     <-- default is 30d, too long
      "maxEntries": 100        <-- default is 500, too large
    }
  }
Note: The trim script removes cron run histories, dead subagents, and old TUI sessions.
  It keeps whatsapp sessions, the main session, and cron job definitions (not their runs).
  Always backup first. Gateway does not need to be restarted after trimming.

### WAL file bloat causing gateway stall (v2026.4.15)
Symptom: Gateway accepts connections but doesn't respond to CLI commands. CPU at 26-76%.
Cause: SQLite WAL files not checkpointing — flows/registry.sqlite-wal grows to 4MB+,
  blocking the Node.js event loop on every read.
Diagnosis: ls -lah ~/.openclaw/flows/ ~/.openclaw/tasks/ — look for large .sqlite-wal files
Fix (gateway must be stopped first):
  systemctl --user stop openclaw-gateway
  pkill -9 -f openclaw
  sqlite3 ~/.openclaw/flows/registry.sqlite "PRAGMA integrity_check; PRAGMA wal_checkpoint(TRUNCATE);"
  sqlite3 ~/.openclaw/tasks/runs.sqlite "PRAGMA integrity_check; PRAGMA wal_checkpoint(TRUNCATE);"
  systemctl --user start openclaw-gateway
Monitor: check WAL sizes periodically. If registry.sqlite-wal regrows to 4MB+, there's
  an underlying checkpoint bug to report to the OpenClaw team.

### Amazon Bedrock plugin causing 117-second startup delay
Symptom: Gateway takes 2+ minutes to start. Logs show long pause between "starting..." and "ready".
Cause: Bedrock plugin does live AWS API model discovery on every startup.
Fix: Disable auto-discovery in ~/.openclaw/openclaw.json:
  { "plugins": { "entries": { "amazon-bedrock": { "config": { "discovery": { "enabled": false } } } } } }
Note (observed April 2026): Even with discovery.enabled=false, startup on this EC2 takes
  135-145 seconds. Something in the 6 active plugins (acpx, browser, device-pair,
  phone-control, talk-voice, whatsapp) is still slow. Not yet identified. Budget 2-3
  minutes for gateway startup on this host after any restart.

### Runaway openclaw-doctor process
Symptom: openclaw-doctor process consuming 70-116% CPU and 300-700MB RAM indefinitely.
Cause: Doctor command spawned from TUI and never completed — becomes orphaned.
Fix: pkill -9 -f openclaw-doctor
Prevention: Do not run openclaw doctor from within a TUI session.

### Gateway auth token stored as placeholder
Symptom: Mac app and TUI get token_mismatch errors after a gateway restart or reinstall.
Cause: openclaw.json gateway.auth.token field contains a placeholder instead of a real token.
  The CLI redacts it with __OPENCLAW_REDACTED__ so it is hard to spot.
Fix: Generate a new token and write it directly to the config:
  python3 -c "import json,secrets; path='/home/ubuntu/.openclaw/openclaw.json'; d=json.load(open(path)); d['gateway']['auth']['token']=secrets.token_hex(16); open(path,'w').write(json.dumps(d,indent=2)); print('done')"
  Then restart gateway and update token on all clients.

### Two openclaw versions co-existing (linuxbrew + system npm)
Symptom: openclaw --version shows different version to what the gateway service runs.
  doctor CLI reports "Config was last written by newer version".
  After update, running `openclaw --version` interactively still shows old version.
Cause: openclaw installed via both linuxbrew and system npm. Interactive shell has
  linuxbrew first in PATH (set by `brew shellenv` in .bashrc), so the old binary wins.
Fix: Remove linuxbrew copy — delete the linuxbrew lib/node_modules/openclaw directory
  and the linuxbrew bin/openclaw symlink:
    rm /home/linuxbrew/.linuxbrew/bin/openclaw
    rm -rf /home/linuxbrew/.linuxbrew/lib/node_modules/openclaw
  Verify: which openclaw (should show /usr/bin/openclaw)
  After removal: if shell still shows old path, run: hash -r
  (bash caches command locations; hash -r clears the cache after removing a binary)

### sessions.json write lock stall (v2026.4.15)
Symptom: Gateway accepts connections but chat.send times out (90-115 seconds).
  Logs show: "[session-write-lock] releasing lock held for 115422ms (max=15000ms)"
  CLI commands time out. TUI shows "disconnected" or "gateway request timeout".
Cause: sessions.json is too large (1MB+). Every agent run rewrites the entire file
  with an exclusive lock. With 50+ entries (cron run histories, dead subagents, old
  TUI sessions) this takes 80-115 seconds and blocks everything else.
Diagnosis: ls -lah ~/.openclaw/agents/main/sessions/sessions.json
  Over ~400KB is a problem. Check entry count:
  python3 -c "import json; d=json.load(open('/home/ubuntu/.openclaw/agents/main/sessions/sessions.json')); print(len(d))"
Fix: Stop gateway, trim sessions.json, restart.
  The safe entries to keep: primary whatsapp sessions, agent:main:main, cron job
  definitions (without :run: in key). Safe to remove: all :run: entries, :subagent:
  entries, old :tui- entries.
  After trimming, tighten maintenance config:
    session.maintenance.pruneAfter: "7d"  (default 30d is too long)
    session.maintenance.maxEntries: 100   (default 500 is too many)
Prevention: Sessions.json grows from cron run histories. Keep maxEntries low and
  pruneAfter short. Also: don't let failed cron jobs accumulate — they add entries fast.

### Cron job pile-up causing memory exhaustion
Symptom: Multiple openclaw child processes accumulate (6-10+), consuming 3-4GB RAM.
  Each child is 300-900MB. Memory pressure causes OOM kills and instability.
Cause: Failed or timed-out cron jobs with stale "runningAtMs" in their state get
  re-spawned on gateway restart. Jobs that use exec with complex inline scripts
  (heredocs, pipelines) fail the 2026.4.15 exec security check and retry repeatedly.
Diagnosis: ps aux | grep openclaw | grep -v grep
  More than 2-3 openclaw processes beyond the gateway = pile-up.
  Check for stale runningAtMs: cat ~/.openclaw/cron/jobs.json | grep runningAtMs
Fix:
  1. Kill child processes (NOT the gateway): ps aux | grep openclaw | grep -v grep |
     grep -v openclaw-gateway | awk '{print $2}' | xargs kill -9
  2. Clear stale lock files: rm -f ~/.openclaw/agents/main/sessions/*.lock
  3. Edit ~/.openclaw/cron/jobs.json directly — remove "runningAtMs" from all job
     state objects, disable any jobs that keep failing
  4. Restart gateway

### exec security change in v2026.4.15 — heredocs blocked
Symptom: Cron jobs or agent runs fail with:
  "exec preflight: complex interpreter invocation detected; refusing to run"
Cause: v2026.4.15 blocks complex shell patterns including heredocs (cat << EOF),
  pipelines to interpreters, and inline script construction. Affects any cron job
  using patterns like: cat << 'PYEOF' > /tmp/script.py ... python3 /tmp/script.py
Fix: Rewrite affected cron job messages to use pre-existing script files:
  Instead of building scripts inline, write the script to disk first (as a file),
  then invoke it with: python3 /path/to/script.py or node /path/to/script.js
  Direct invocations (python3 file.py, node file.js, bash script.sh) are allowed.

### TUI "disconnected" after gateway URL change
Symptom: openclaw tui shows disconnected immediately after changing gateway.remote.url
Cause: Mac OpenClaw app or TUI has the old URL cached. Or the connection flood from
  a previous token_mismatch loop filled the gateway handshake queue.
Fix: Kill child processes on EC2 to free the queue (see cron pile-up fix above),
  wait 30s, then retry. Or pass URL explicitly: openclaw tui --url wss://hostname

### Web UI returns empty page when accessed via /openclaw Caddy route

Symptom: Navigating to https://rocky.onefellswoop.com.au/openclaw returns a blank page.
  Direct access to http://127.0.0.1:18789/ works fine (HTTP 200, full HTML).
Cause: The Caddyfile routes /openclaw* to port 18789 without stripping the prefix. The
  gateway web UI serves from / with relative asset paths (./favicon.svg, ./manifest.webmanifest
  etc). Loading via /openclaw prefix breaks all asset resolution — blank page results.
  The /openclaw route exists for the Mac app WebSocket connection (wss://), NOT for the
  web UI HTML.

Fix (Caddy prefix strip — add a dedicated route):
  handle /webchat* {
      uri strip_prefix /webchat
      reverse_proxy 127.0.0.1:18789
  }

Workaround (immediate): Access http://localhost:18789 directly in Mac browser — the SSH
  tunnel already forwards localhost:18789 to the EC2 gateway. Works with no Caddy involved.
  Or: use the web UI via the Mac App directly.

Note: Token is still required when accessing the web UI. Enter it in the gateway's login
  prompt. The gateway auth token is in ~/.openclaw/openclaw.json at gateway.auth.token.

### Mac TUI input accepted but not rendered for minutes (remote mode latency)

Symptom: Mac TUI shows "connected" and a cursor appears in the input area. Typing produces
  no visible text, or text appears in a batch after several minutes.
Cause: Mac TUI is connecting in remote mode via wss://rocky.onefellswoop.com.au/openclaw
  (the public Caddy URL). Every keystroke round-trips: Mac → Caddy (EC2) → gateway → back
  to Mac to render. On any variable internet connection this produces exactly the described
  symptom — input buffers locally, renders in a burst when network catches up.
  This is DIFFERENT from a TUI hang/freeze. The TUI is working; the latency is the problem.

Fix: Set tui.url to use the SSH tunnel (ws://127.0.0.1:18789) which is already running
  on the Mac (ai.openclaw.gateway-tunnel LaunchAgent). The tunnel carries keystrokes over
  the existing SSH connection — zero added latency.

  python3 -c "
  import json
  path = '/Users/paulm/.openclaw/openclaw.json'
  d = json.load(open(path))
  d['tui'] = {'url': 'ws://127.0.0.1:18789'}
  open(path, 'w').write(json.dumps(d, indent=2))
  print('done')
  "

  Then close and reopen the TUI. No gateway restart needed.

Verify tunnel is up before applying fix:
  curl -s -o /dev/null -w "%{http_code}" --max-time 5 http://127.0.0.1:18789/
  (should return 200)

Note: The gateway.remote.url (wss://rocky.onefellswoop.com.au/openclaw) should stay as-is
  for the Mac app and node host. Only the tui section needs this override.

### Mac TUI "connecting..." then hangs forever — gateway.remote.url wrong (confirmed May 2026)

Symptom: TUI comes up, shows "connecting...", then just hangs at a blank TUI screen.
  No error message. No Ctrl-C response (terminal is frozen in alternate-screen mode).
  EC2 gateway logs show ZERO incoming connection attempts from the Mac.
  Mac node (LaunchAgent) may connect fine — you can see res ok calls in EC2 logs — but
  the TUI is a separate process and uses a different URL.

Root cause: Mac's gateway.remote.url was set to ws://localhost:18789 — pointing at the
  Mac itself. There is no gateway on the Mac. The TUI connects to that address, gets
  nothing, and hangs forever.

This is DIFFERENT from the EC2-side TUI hang (zombie processes starving event loop).
The Mac TUI hangs before even opening a socket to EC2 — the gateway never sees it.

Diagnosis — check what URL the Mac is using:
  cat ~/.openclaw/openclaw.json | python3 -c "import json,sys; c=json.load(sys.stdin); print(c.get('gateway',{}).get('remote',{}))"
  Look for: url field. If it says ws://localhost:18789 or similar — that is the bug.

Fix:
  openclaw config set gateway.remote.url wss://rocky.onefellswoop.com.au/openclaw
  openclaw gateway restart

IMPORTANT — the correct URL for Paul's Mac is:
  wss://rocky.onefellswoop.com.au/openclaw
  (Caddy on the EC2 routes /openclaw* to 127.0.0.1:18789)

Note: An SSH local-forward tunnel (ssh -N -L 18789:127.0.0.1:18789) runs as a KEEPALIVE
  LaunchAgent (ai.openclaw.gateway-tunnel) on the Mac. This tunnel IS intentional and
  needed — OpenClaw uses it for various functions beyond the TUI. Do NOT disable or kill it.
  The tunnel process will show "Connection refused" errors in /tmp/openclaw-gateway-tunnel.log
  during EC2 reboots — this is normal; the KeepAlive reconnects automatically once EC2 is back.
  The exit code 255 in `launchctl list` after a reboot is from a failed reconnect attempt
  during the EC2 downtime — not a fault. Verify it recovered with:
    lsof -i :18789  (should show ssh process LISTENING + ESTABLISHED connections)
    curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:18789/  (should return 200)
  Do NOT confuse it with any reverse tunnel used for workspace sync.

NOT wss://rocky-on-ec2.tail9c7933.ts.net — MagicDNS is OFF on Paul's Mac.
NOT wss://rocky-on-ec2.tail9c7933.ts.net:18789 — port 18789 is loopback-only on EC2.
NOT ws://localhost:18789 — there is no gateway on the Mac.

After fix: Mac LaunchAgent (node host) reconnects to EC2. Verify by checking EC2 logs
  for a burst of res ok calls (health, system-event, device.pair.list, node.list, config.get).

### Mac app sessions.list poll causing 17-second event loop freeze

Symptom: Gateway logs show sessions.list taking 10,000+ ms. Immediately followed by
  liveness warning: eventLoopDelayMaxMs=17649.6 eventLoopUtilization=0.76
  TUI handshake attempts during this window all fail (15s budget exceeded).
  Occurs even with no agent work (active=0, waiting=0, queued=0).
Cause: Mac app is connected and polling sessions.list. This call triggers a
  sessions.json read/write under exclusive lock. If sessions.json has grown at all
  (even 276KB / 8 entries can be slow under load) the lock blocks the event loop
  for the duration, preventing any TUI handshake from completing.
  The Mac app staying connected while troubleshooting actively makes things worse —
  every poll it sends triggers another event loop freeze.
Diagnosis:
  Watch for: "sessions.list NNNNNms" entries in gateway logs — anything over 2000ms
  is a warning sign. Followed by eventLoopDelayMaxMs spikes of similar duration.
Fix (immediate): Disconnect Mac app from gateway while troubleshooting TUI connection.
  Or trim sessions.json (see sessions.json bloat section).
Fix (permanent): Keep sessions.json small via maintenance config and regular trimming.
Note: This is distinct from the sessions.json WRITE lock stall — this is the READ
  path being slow due to file size and lock contention. Both paths go through the
  same exclusive lock mechanism.

### Mac app shows "2 instances of the gateway" / presence unavailable decode failed

Symptom: Mac app intermittently shows "2 instances of the gateway" and
  "Presence unavailable (decode failed); showing health probe + local fallback".
  App connects then drops repeatedly. Chat messages get no response.
Cause: SSH local-forward tunnel (ssh -N -L 18789:127.0.0.1:18789) is still running
  alongside the wss:// Caddy connection. The Mac app sees both paths to the same
  EC2 gateway, causing decode conflicts. Both mac app and/or mac CLI were
  previously configured to use ws://localhost:18789 routed through the tunnel.
  After migrating CLI to wss://rocky.onefellswoop.com.au/openclaw, the tunnel
  becomes redundant but the process (ssh -N -L ...) keeps running.
Diagnosis:
  ps aux | grep openclaw | grep -v grep
  Look for: /usr/bin/ssh -N -L 18789:127.0.0.1:18789 ... ubuntu@<ec2-ip>
  This is a LOCAL forward (-L), not a reverse tunnel (-R). Harmless once both app
  and CLI are on wss:// but confirms tunnel is still active.
Fix: Kill the tunnel process by PID. Or leave it running (harmless) once both
  app and CLI are confirmed pointing to wss://rocky.onefellswoop.com.au/openclaw.
  The tunnel was the original connection mechanism before Caddy was configured.
  It is now superseded by the wss:// URL and can be safely removed.
Note: Do NOT mistake -L (local forward) for -R (reverse tunnel). The reverse tunnel
  in this setup runs FROM EC2 to Mac for rsync/SSH access. The -L tunnel runs
  FROM Mac to EC2 for gateway access and is the one that is now redundant.

### Mac gateway LaunchAgent (ai.openclaw.gateway) must not run

Symptom: Mac app shows 2 gateway instances, repeated disconnects, "decode failed".
Cause: The Mac gateway LaunchAgent (ai.openclaw.gateway) was running alongside
  the EC2 gateway. The Mac should only run as a NODE host, not a gateway.
Fix (already applied to this setup, kept for reference if it recurs):
  launchctl stop ai.openclaw.gateway
  launchctl disable gui/$(id -u)/ai.openclaw.gateway
  launchctl unload ~/Library/LaunchAgents/ai.openclaw.gateway.plist
Verify:
  launchctl list | grep openclaw
  # Should only show node service, not gateway
Note: disabling the LaunchAgent stops future auto-starts but does NOT kill the
  currently running process. Check ps aux for any running openclaw gateway
  process on the Mac and kill it if found.

### 2026.4.29 agent prep 73-100s per turn — event loop starvation (confirmed May 2026)

Symptom: Web UI chat bubble appears briefly then disappears with no reply. Agent turns take
  2-5 minutes. Gateway logs show prep stages totalMs=73000-100000ms:
    system-prompt:23000ms, stream-setup:20000ms, core-plugin-tools:9000-19000ms
  eventLoopDelayMaxMs spikes to 34000ms (34 seconds). 408 fetch-timeout log lines per 2h window.
  Turning off Telegram does NOT fix this — it is a 4.29 regression unrelated to channels.
Cause: 4.29 substantially reworked the dispatch flow (9 prepStages.mark hits vs 0 in 4.27).
  synchronous CPU-bound work blocks the Node.js event loop on every agent turn:
  - system-prompt rebuild not cached (runs fully on every turn)
  - stream-setup overhead high
  - core-plugin-tools factory invoked for every plugin on every run (~10-19s)
  - ACPX plugin installs ~42 npm deps synchronously on first use
Root cause fixed on main (ACPX deferred, scoped tool discovery, registry reuse) but NOT
  released in any npm version as of May 2026.
Fix: Downgrade to openclaw@2026.4.23 (stable, predates all these regressions).
  See: GitHub issues #75999, #75290, #75984

### 2026.4.29 Web UI chat bubble disappears silently — OpenAI transport bug (confirmed May 2026)

Symptom: Web UI loads, message sent, assistant bubble appears briefly then vanishes. No reply.
  Direct OpenAI API works fine on same machine. Gateway logs show:
    "produced no reply before the idle watchdog"
    "embedded run failover decision ... reason=timeout"
    "FailoverError: LLM request timed out"
  OpenAI platform shows tokens were consumed but OpenClaw records assistantTexts: []
Cause: OpenClaw's OpenAI Responses WebSocket stream path silently fails to surface the reply.
  The Control UI compounds this by clearing the chat stream on error without showing a message.
  This is an OpenAI-specific transport bug in 4.29. Anthropic transport works fine.
Workaround (immediate): Change primary model to anthropic/claude-sonnet-4-6:
  python3 -c "
  import json
  path = '/home/ubuntu/.openclaw/openclaw.json'
  d = json.load(open(path))
  d['agents']['defaults']['model']['primary'] = 'anthropic/claude-sonnet-4-6'
  d['agents']['defaults']['model']['fallbacks'] = ['openai/gpt-5.4', 'openai/gpt-5.4-mini', 'groq/llama-3.3-70b-versatile']
  open(path, 'w').write(json.dumps(d, indent=2))
  print('done')
  "
  No gateway restart needed — config picked up on next turn.
Alternative workaround: Set transport: "sse" explicitly in OpenAI provider config.
Fix: Downgrade to openclaw@2026.4.23. See: GitHub issue #75824

### 2026.4.29 TUI deadlock — multiple root causes (confirmed May 2026)

Symptom: openclaw tui hangs at "connecting | idle", never connects, unresponsive to Ctrl+C.
  Gateway shows "handshake timeout" / "connect challenge timeout code=1008" — TUI connects
  at TCP level but never sends auth frame.
Confirmed distinct root causes in 4.29 (GitHub issues #75844, #75379, #75717, #75981):
  1. TUI process event loop blocks before sending auth — async context setup issue
  2. Stale dist files after npm update (references missing control-ui-*.js hash)
  3. model normalization during context window warmup triggers hundreds of hook calls (100% CPU)
  4. TUI unresponsive to Ctrl+C after WS close — drainInput() has no timeout bounds
None of these are fixed in the current npm release (still 4.29 as of May 2026).
OPENCLAW_NO_RESPAWN=1 + explicit --url and --token partially bypasses issue 1 but not others.
Fix: Downgrade to openclaw@2026.4.23. Web UI (localhost:18789) works as TUI substitute.
See: GitHub issues #75844, #75379, #75717, #75981, PR #75381

### Rocky's SSH probing leaves zombie TUI processes — CRITICAL pitfall (May 2026)

When Rocky (Hermes) SSH-probes the EC2 gateway using OPENCLAW_NO_RESPAWN=1 openclaw tui
  in background processes, each probe leaves a zombie openclaw-tui process running at
  60-90% CPU. These accumulate silently and can consume 3GB+ RAM and 200%+ CPU within
  minutes, starving the gateway event loop and making the entire system unresponsive.
The TERM=dumb environment from Rocky's SSH sessions means TUI probes can never actually
  connect anyway — they are always wasted. Never run openclaw tui from Rocky's SSH sessions.
Detection: ps aux | grep openclaw | grep -v grep — look for multiple openclaw-tui PIDs
  with high CPU and growing TIME+. Gateway RAM jumping from ~500MB to 4GB+ is a signal.
Cleanup: kill -9 <all openclaw-tui PIDs> — RAM drops immediately.
Rule: Rocky must NEVER run openclaw tui to probe gateway health. Use gateway probe,
  journalctl log inspection, and HTTP checks (curl http://127.0.0.1:18789/) instead.

### Web UI accessible via localhost:18789 directly (confirmed May 2026)

The gateway serves its full web UI at http://127.0.0.1:18789/ (HTTP 200, full HTML).
  This works as a TUI substitute when TUI is deadlocked.
The Caddyfile routes /openclaw* to 18789 but the web UI uses relative asset paths,
  so loading it via the /openclaw prefix breaks all asset loads → empty page.
Workaround to access via Caddy: Add a strip_prefix rewrite in Caddyfile:
  handle /webchat* {
      uri strip_prefix /webchat
      reverse_proxy 127.0.0.1:18789
  }
For local access: Mac tunnel (ssh -L 18789:...) already forwards localhost:18789 on Mac.
  Open http://localhost:18789 in Mac browser → full web UI, token entry prompt, works.
The gateway requires auth token entry on first load (token from openclaw.json gateway.auth.token).

### Mac TUI "gateway disconnected" with MagicDNS disabled
Fix: Use wss://rocky.onefellswoop.com.au/openclaw (public DNS via Caddy).

### 2026.4.29 — AVOID on this setup (May 2026)
Multiple open regressions: 73-100s agent prep (#75999), TUI deadlock (#75844), OpenAI transport
silent failure (#75824), Telegram IPv6 timeouts (#75539). All fixes on main, none released.
Stable version: 2026.4.23. Downgrade: sudo npm install -g openclaw@2026.4.23 (EC2),
/opt/homebrew/bin/npm install -g openclaw@2026.4.23 (Mac homebrew).

### Mac "control ui requires device identity" on --url tunnel override
Using openclaw tui --url ws://127.0.0.1:18789 fails even via SSH tunnel — gateway rejects
control-ui over non-HTTPS ws://. Fix: use OPENCLAW_NO_RESPAWN=1 without --url override:
  alias openclaw-tui="OPENCLAW_NO_RESPAWN=1 openclaw tui"  (in /Users/paulm/.zshrc)

### "tui" root config key rejected by 4.23
Key added after 4.23 — causes "Unrecognized key: tui" error. Remove it; use --url/--token flags.

### ServBay PATH conflict hides homebrew openclaw (Mac)
/Applications/ServBay/package/node/current/bin/ precedes homebrew in login PATH.
Diagnosis: bash -l -c 'which -a openclaw' — ServBay path first = conflict.
Fix: /Applications/ServBay/package/node/current/bin/npm uninstall -g openclaw
  LaunchAgents hardcode /opt/homebrew paths and are unaffected.

### gateway.remote.url keeps reverting to port :18789
Symptom: After setting remote.url, openclaw tui header shows wss://...:18789 again.
Cause: Something (app, node-host process, or a restart) is rewriting openclaw.json
  with a port appended. The Mac app in particular may override config on launch.
Fix: Set the URL via direct python3 edit (not openclaw config set) so it's exactly
  right, then immediately restart the affected LaunchAgent so the app reads the new
  value before it can overwrite it.

### RPC probe timeout on EC2 (openclaw gateway probe hangs)
Symptom: timeout 5 openclaw gateway probe times out. timeout 10 also fails.
Cause: Node.js CLI startup on EC2 takes 3-5 seconds. Default probe timeout too short.
Fix: Use timeout 30 openclaw gateway probe — RPC works fine, just needs 30s budget.
Note: NODE_COMPILE_CACHE helps (openclaw --version in 0.29s) but probe itself needs
  more time to connect and complete the RPC handshake.

### Internal sqlite-wal-checkpoint cron causing high CPU + RPC timeouts
Symptom: openclaw gateway probe times out. ps aux shows 2 child openclaw processes
  at 100-150% CPU. Coincides with every 30-minute wal checkpoint cron run.
Cause: The cron runs as a full agent turn, spawning 2 child processes. These hold
  the sessions.json write lock for 2+ minutes, blocking the gateway event loop.
  The cure (checkpoint cron) was causing the symptom (RPC timeout).
Fix: Disable the internal cron and replace with a system-level cron:
  1. python3 -c "import json; p='/home/ubuntu/.openclaw/cron/jobs.json'; d=json.load(open(p)); [j.update({'enabled':False}) for j in (d if isinstance(d,list) else d.get('jobs',[])) if j.get('name')=='sqlite-wal-checkpoint']; open(p,'w').write(json.dumps(d,indent=2))"
  2. Create ~/scripts/wal-checkpoint.sh (PASSIVE checkpoint, safe while gateway runs)
  3. Add to crontab: */15 * * * * /home/ubuntu/scripts/wal-checkpoint.sh
WAL checkpoint script content:
  #!/bin/bash
  sqlite3 ~/.openclaw/tasks/runs.sqlite "PRAGMA wal_checkpoint(PASSIVE);"
  sqlite3 ~/.openclaw/flows/registry.sqlite "PRAGMA wal_checkpoint(PASSIVE);"

### Mac node TUI shows "gateway disconnected: closed | idle" (mode=remote setup)
Symptom: openclaw tui opens but immediately shows "gateway disconnected: closed | idle".
  EC2 logs show no incoming connection attempt at all.
Cause (1): Mac config has gateway.remote.url set to wss://<tailscale-ip>:18789 — wrong.
  Port 18789 on EC2 is loopback-only. Tailscale serve proxies via HTTPS on port 443.
  Correct URL has no port: wss://rocky-on-ec2.tail9c7933.ts.net
Cause (2): Mac's local openclaw-gateway LaunchAgent is crash-looping with:
  "Gateway start blocked: set gateway.mode=local (current: remote)"
  When mode=remote, the local gateway refuses to start. If the LaunchAgent keeps trying,
  it fills logs every 5 seconds but never provides a local WS endpoint for the TUI.
  The TUI connects to the local gateway by default — so if there's no local gateway,
  it shows disconnected even though the EC2 is fine.
Cause (3): openclaw --url flag requires --token when overriding URL explicitly.
  Error: "gateway url override requires explicit credentials"
  Fix: openclaw tui --url wss://hostname --token <token>
IMPORTANT: Disabling or unloading the Mac gateway LaunchAgent does NOT kill the currently
  running gateway process. After disabling, always also kill the process explicitly:
    pkill -f "openclaw.*gateway"
  Failure to do this leaves a ghost gateway running. The Mac app will then see BOTH the
  ghost local gateway AND the EC2 gateway simultaneously — showing "2 instances" in the app
  and "Presence unavailable (decode failed); showing health probe + local fallback".

Fix (full sequence for mode=remote Mac node):
  1. Fix the remote URL (no port): set gateway.remote.url = wss://rocky-on-ec2.tail9c7933.ts.net
     python3 -c "import json; path='/Users/paulm/.openclaw/openclaw.json'; d=json.load(open(path)); d.setdefault('gateway',{}).setdefault('remote',{})['url']='wss://rocky-on-ec2.tail9c7933.ts.net'; open(path,'w').write(json.dumps(d,indent=2))"
  2. Stop the crash-looping local gateway LaunchAgent:
     launchctl stop gui/$(id -u)/ai.openclaw.gateway
     launchctl unload ~/Library/LaunchAgents/ai.openclaw.gateway.plist
     pkill -f "openclaw/dist/index.js gateway"
  3. Now openclaw tui connects directly to EC2 (mode=remote routes TUI to remote gateway).
     If MagicDNS doesn't resolve, use explicit URL + token:
     openclaw tui --url wss://rocky-on-ec2.tail9c7933.ts.net --token <token>
Note: The Mac app (OpenClaw.app) connects independently of the TUI and LaunchAgent.
  It has its own token setting in app Preferences > Gateway. Update it separately.
  App connection success shows as res ✓ calls in EC2 gateway logs.
Note: "openclaw" binary not in PATH when SSH'd into Mac via non-login shell.
  Use full path: /opt/homebrew/bin/openclaw, or run commands in a login shell.

### Token mismatch after EC2 or Mac reboot (recurring pattern)
Symptom: Mac app and TUI both show disconnected after any reboot. EC2 logs flood with
  "reason=token_mismatch" from "Paul's M5 Max MacBook Pro" for both role=operator (app)
  and role=node (CLI). Two distinct clients, both failing.
Cause: After reboot the Mac loses the auth token from memory. The EC2 gateway still has
  the correct token in openclaw.json, but the Mac clients reconnect with a stale token.
Fix sequence:
  1. Generate new token on EC2:
     python3 -c "import json,secrets; path='/home/ubuntu/.openclaw/openclaw.json'; d=json.load(open(path)); tok=secrets.token_hex(16); d['gateway']['auth']['token']=tok; open(path,'w').write(json.dumps(d,indent=2)); print('NEW_TOKEN:'+tok)"
  2. Restart EC2 gateway to pick up new token:
     systemctl --user restart openclaw-gateway
  3. Update Mac CLI token:
     openclaw config set gateway.auth.token <new-token>  (run on Mac)
  4. Update Mac app token: OpenClaw app > Preferences/Settings > Gateway > auth token
  5. Restart Mac LaunchAgent to pick up new token:
     launchctl stop gui/$(id -u)/ai.openclaw.gateway
     launchctl start gui/$(id -u)/ai.openclaw.gateway
Note: Kill any orphaned openclaw child processes on EC2 before restarting gateway —
  after a reboot they accumulate fast from token_mismatch reconnection floods.
  ps aux | grep openclaw | grep -v grep | grep -v openclaw-gateway | awk '{print $2}' | xargs kill -9

### Post-reboot health check sequence (EC2 + Mac)

After any EC2 reboot, run this sequence before attempting TUI:

  1. On EC2 (via SSH):
     which node && node --version           # must show /usr/bin/node v24.15.0
     ps aux | grep openclaw | grep -v grep  # gateway only, no zombies
     systemctl --user status openclaw-gateway --no-pager | head -20
     journalctl --user -u openclaw-gateway --no-pager -n 20
     ls -lah ~/.openclaw/agents/main/sessions/sessions.json
     ls -lah ~/.openclaw/flows/*.sqlite* ~/.openclaw/tasks/*.sqlite*

  2. On Mac (via EC2 → ssh paulm@100.87.144.6):
     ps aux | grep openclaw | grep -v grep          # no zombie TUI processes
     lsof -i :18789                                 # tunnel listening + ESTABLISHED
     curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:18789/   # 200 = tunnel ok
     python3 -c "import json; d=json.load(open('/Users/paulm/.openclaw/openclaw.json')); print(d.get('gateway',{}).get('remote',{}))"
     # url must be wss://rocky.onefellswoop.com.au/openclaw

  3. Expected healthy state after clean reboot:
     - Gateway: single process, CPU <20%, no event loop warnings after first 60s
     - Node: /usr/bin/node v24.15.0 (PATH fix in .bash_profile survives reboot)
     - Sessions.json: <200KB
     - WAL files: no .sqlite-wal files, or small (<100KB)
     - Mac tunnel: LISTEN + ESTABLISHED connections, HTTP 200
     - Mac node hosts: 2 openclaw-node processes connected via tunnel
     - No zombie TUI processes on either machine

  4. Normal post-reboot noise to IGNORE:
     - "Connection refused" entries in /tmp/openclaw-gateway-tunnel.log (tunnel reconnecting)
     - exit code 255 in `launchctl list | grep openclaw.gateway-tunnel` (failed reconnect during EC2 down)
     - Event loop spike (P99 ~12s) in first 30s window of gateway logs (Telegram init)
     - "menu text exceeded ... payload budget" Telegram warnings
     - "fetch timeout reached" during Telegram startup

### Remote Mac access path (Rocky → OpenClaw EC2 → Mac)

When SSH'd into OpenClaw EC2, the Mac is reachable via Tailscale:
  ssh paulm@100.87.144.6
  (No key needed — EC2 already has trust to Mac via Tailscale)
This path works even when there is no reverse tunnel from EC2 to Mac.
Use this to check Mac OpenClaw state, kill zombie TUI processes, or inspect config
while Paul is away from the Mac (e.g. working remotely from iPad).

### Mac node Tailscale DNS not resolving rocky-on-ec2.tail9c7933.ts.net
Symptom: SSH to rocky-on-ec2.tail9c7933.ts.net fails with "cannot resolve hostname".
  Occurs after Mac reboot if Tailscale MagicDNS is slow to start.
Fix (immediate): Use Tailscale IP directly — 100.100.35.63 — no DNS needed.
  In OpenClaw app SSH settings: ubuntu@100.100.35.63
Fix (permanent): Ensure Tailscale MagicDNS is enabled in Mac Tailscale app Preferences.
Note: The Tailscale IP (100.100.35.63) is stable and can be used everywhere instead of
  the hostname. Port 18789 must NOT be used — Tailscale serve proxies via 443 (HTTPS).
  Correct URL: wss://rocky-on-ec2.tail9c7933.ts.net (no port)
  Wrong URL:   wss://100.100.35.63:18789 (port 18789 is loopback-only on EC2)

### gateway.remote.url must use wss:// for non-loopback (v2026.4.15)
Symptom: "SECURITY ERROR: Gateway URL ws://x.x.x.x:18789 uses plaintext ws:// to
  a non-loopback address"
Cause: v2026.4.15 enforces WSS for any non-localhost gateway URL.
Fix: openclaw config set gateway.remote.url wss://your-tailscale-hostname
  Use Tailscale hostname (wss://rocky-on-ec2.tail9c7933.ts.net) not raw IP.
  Raw IP with ws:// only works if OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1 is set.

### Mac gateway restart hangs (health check timeout)
Symptom: openclaw gateway restart hangs for 60s then reports "timed out waiting
  for health checks"
Cause: Mac LaunchAgent gateway can't pass health checks if the remote EC2 gateway
  is unresponsive or the SSH tunnel is broken.
Fix: pkill -9 -f openclaw-gateway on Mac, then openclaw gateway start
  Or skip the Mac gateway entirely — the TUI connects directly to the EC2 via
  wss:// Tailscale URL and doesn't need the Mac gateway running.

### systemctl restart kills gateway mid-boot (SIGTERM on slow startup)
Symptom: `systemctl --user restart openclaw-gateway` times out (default 20s systemd
  timeout), which sends SIGTERM to the gateway while it is still starting up (startup
  takes 135-145s on this EC2). The gateway gets killed, systemd restarts it again,
  and you end up in a loop of killed-mid-boot cycles.
Cause: systemd's default TimeoutStartSec is shorter than the gateway's startup time.
Fix: Use stop + start separately, NOT restart. Stop first (allow it to shut down cleanly),
  then start and wait a full 2-3 minutes before checking if the port is up:
    systemctl --user stop openclaw-gateway
    sleep 5
    systemctl --user start openclaw-gateway
    # wait ~150 seconds then check: ss -tlnp | grep 18789
  Alternatively, increase the timeout in the service override:
    mkdir -p ~/.config/systemd/user/openclaw-gateway.service.d/
    echo -e "[Service]\nTimeoutStartSec=300" > ~/.config/systemd/user/openclaw-gateway.service.d/timeout.conf
    systemctl --user daemon-reload
  NEVER use `systemctl --user restart` on this EC2 — it will SIGTERM the gateway mid-boot.

### sessions.json write lock stalls (80-115 seconds)
Symptom: Gateway responsive but CLI commands and WhatsApp queue for 80-115 seconds.
  Log shows: [session-write-lock] releasing lock held for 115422ms (max=15000ms)
Cause: sessions.json grown large (1.8MB, 59 entries) with accumulated cron run histories,
  dead subagents, and old TUI sessions. Every agent write holds exclusive lock while
  rewriting the entire file.
Diagnosis:
  ls -lah ~/.openclaw/agents/main/sessions/sessions.json  (watch for >500KB)
  ps aux | grep openclaw | grep -v grep  (watch for 6+ child processes)
Quick fix: kill child agent processes (not the gateway), clear lock files:
  ps aux | grep openclaw | grep -v grep | grep -v openclaw-gateway | awk '{print $2}' | xargs kill -9
  rm -f ~/.openclaw/agents/main/sessions/*.lock
Permanent fix: trim sessions.json (remove cron :run: entries, old subagents, old TUI sessions).
  Back up first, then edit directly with python3. Keep: whatsapp sessions, main session,
  cron job definitions (not their run histories).
Prevention: set session.maintenance.pruneAfter: "7d" and maxEntries: 100 in config.
  Also clear stale runningAtMs from cron jobs.json after killing child processes.

### buc-dashboard EADDRINUSE on port 3744 after reboot

Symptom: buc-dashboard.service enters crash-loop immediately after EC2 reboot.
  journalctl shows: Error: listen EADDRINUSE: address already in use :::3744
  Service status: activating (auto-restart) with rapid exit-code failures.
Cause: Another service (likely studio-dashboard or a dashboard sibling) grabbed port 3744
  before buc-dashboard started. Race condition on boot ordering.
Diagnosis: ss -tlnp | grep 3744  (find what process holds it)
  systemctl --user list-units | grep dashboard  (check which dashboards are up)
Fix: Stop the service holding 3744, let buc-dashboard restart, or add After= dependency
  in the buc-dashboard service unit to start after its sibling services.
Note: This does NOT affect the OpenClaw gateway — it's a dashboard service issue only.
  The gateway runs on 18789 and is unaffected.

### SSH local-forward tunnel (Mac → EC2) creates phantom "2 instances" in Mac app

Symptom: Mac app shows 2 gateway instances intermittently. "Presence unavailable (decode failed);
  showing health probe + local fallback." App connects but drops repeatedly, no response to chat.
  An SSH tunnel process is visible: /usr/bin/ssh -N -L 18789:127.0.0.1:18789 ... ubuntu@<ec2-ip>

Cause: The tunnel was set up when the Mac CLI used ws://localhost:18789 (routed via tunnel to EC2).
  After migrating the Mac to wss://rocky.onefellswoop.com.au/openclaw, the tunnel became redundant
  but kept running. The Mac app now connects via BOTH the wss:// URL and the tunnel endpoint
  on localhost:18789, both routing to the same EC2 gateway — which appears as 2 instances.

Fix: Kill the tunnel process:
  ps aux | grep "ssh.*-L 18789" | grep -v grep
  kill -9 <pid>

Note: The tunnel is NOT the mac node LaunchAgent (ai.openclaw.node.plist). It is a separate
  process, likely started manually or by an older launchd entry. Safe to kill once Mac CLI
  and app are both using wss://rocky.onefellswoop.com.au/openclaw directly.

Architecture note: The original setup used the tunnel because the Mac pointed to localhost:18789.
  Current setup: Mac CLI and app both use wss://rocky.onefellswoop.com.au/openclaw directly,
  tunnelled via Caddy on the EC2. The local-forward tunnel is no longer needed.

### PATH fix must go in BOTH .bashrc AND .bash_profile (login shells)

Symptom: `source ~/.bashrc && which node` shows correct v24.15.0 in current session,
  but new SSH sessions immediately revert to linuxbrew node v25.9.0.
Cause: SSH sessions are LOGIN shells — they read .bash_profile, not .bashrc.
  The PATH fix appended to .bashrc only applies to interactive non-login shells.
Fix: Add the PATH override to BOTH files:
  echo 'export PATH="/home/ubuntu/.local/bin:$PATH"  # prefer system node over linuxbrew' >> ~/.bash_profile
  echo 'export PATH="/home/ubuntu/.local/bin:$PATH"  # prefer system node over linuxbrew' >> ~/.bashrc
Important: Each new SSH session that opens because a previous one hung will use the
  login shell PATH — so every new session needs node v24 from the start. Without
  .bash_profile fix, every TUI attempt in a new session uses v25 and creates a zombie.


### Zombie openclaw-tui processes starving gateway event loop (May 2026)

Symptom: TUI says "connecting..." then immediately "gateway connect failed: Error: gateway closed (1000):".
  EC2 gateway logs show "handshake timeout" — TUI connects at TCP level but never sends auth frame.
  Every connection attempt times out at the gateway side (15-second handshake budget exceeded).
  Gateway HTTP 200 on /. The gateway itself is running fine.

Cause: Earlier TUI hang sessions left openclaw-tui processes still running in the background.
  Each spins at 90%+ CPU indefinitely — they never clean up after a failed connect.
  Two such zombies can consume 180%+ CPU, completely starving the gateway's Node.js event loop.
  When the event loop is starved, the gateway cannot complete any WebSocket handshakes —
  even though port 18789 is open and accepting TCP connections.

CRITICAL: This problem occurs on BOTH EC2 AND MAC. When the Mac TUI hangs, it also leaves
  zombie openclaw-tui processes on the Mac that must be killed separately.
  Check BOTH machines when TUI is failing.

Diagnosis — do this FIRST on BOTH machines when TUI fails to connect:
  ps aux | grep openclaw | grep -v grep
  Look for: 'openclaw-tui' processes (distinct from 'openclaw-gateway').
  Warning signs: any openclaw-tui at 70%+ CPU, especially with long TIME+ values.
  If you see 2+ openclaw-tui processes from earlier sessions, that IS the problem.

Fix (EC2):
  pkill -9 -f openclaw-tui
  # Verify all gone:
  ps aux | grep openclaw-tui | grep -v grep
  # Wait 30-60 seconds for load average to drop, then retry TUI.

Fix (Mac):
  kill -9 <pid1> <pid2>   # use specific PIDs from ps aux output
  # pkill -9 -f openclaw-tui also works on Mac

Note: The terminal session that spawned the hung TUI will be frozen (raw/alternate-screen mode).
  Close it or type `reset` after killing the process to restore terminal.
  On Mac: each new SSH session opened because a previous one hung is a new login shell.
  After killing zombies, source ~/.bashrc or check `which node` before retrying TUI —
  login shells may have linuxbrew node in PATH (see PATH fix note above).

Prevention: Before each TUI session, check for leftover processes. After any TUI freeze,
  immediately kill the background process — don't just close the terminal window.
  Also ensure .bash_profile has the node PATH fix so new login shells start with v24.
  # Identify openclaw-tui PIDs at high CPU
  kill -9 <pid1> <pid2>
  # Wait 30-60 seconds for load average to drop, then retry TUI.

Note: The terminal session that spawned the hung TUI will be frozen (raw/alternate-screen mode).
  Close it or type `reset` after killing the process to restore terminal.

Prevention: Before each TUI session, check for leftover processes on BOTH hosts.
  After any TUI freeze, immediately kill the background process — don't just close the terminal window.

### Linuxbrew node version conflict causing TUI/CLI hang after update (May 2026)

Symptom: After updating openclaw, running `openclaw tui` from an interactive SSH session
  shows "connecting..." then freezes completely — no keypress response, no Ctrl-C.
  `openclaw --version` works (flag-only, no gateway contact), but any connected command
  (tui, status, doctor) hangs indefinitely or times out.
  EC2 gateway logs show "handshake timeout" — the TUI never sends auth.
  In logs: TUI process is running with runtimeVersion 25.9.0 while gateway uses 24.15.0.

Cause: This EC2 has two Node installs:
  - /usr/bin/node (v24.15.0) — system npm, where openclaw is installed
  - /home/linuxbrew/.linuxbrew/bin/node (v25.9.0) — linuxbrew
  The openclaw.mjs shebang is #!/usr/bin/env node. Interactive shell has linuxbrew
  first in PATH (from `brew shellenv` in .bashrc), so linuxbrew node v25.9.0 runs the
  TUI. 2026.4.29 TUI startup has a compatibility issue with Node 25 — the Ink UI
  enters alternate-screen mode but spins indefinitely in the JS event loop.

Important: `openclaw gateway status` worked fine via non-interactive SSH (system node,
  no linuxbrew PATH) but hung when run as a subprocess from interactive shell sessions.
  The hang is not about the gateway — it's the CLI process itself.

Fix: Create a ~/.local/bin/node symlink pointing to /usr/bin/node (v24), then ensure
  ~/.local/bin comes before linuxbrew in the interactive shell PATH.

  Step 1: Create symlink
    mkdir -p ~/.local/bin
    ln -sf /usr/bin/node ~/.local/bin/node

  Step 2: Add to end of BOTH ~/.bashrc AND ~/.bash_profile (AFTER the brew shellenv line):
    export PATH="/home/ubuntu/.local/bin:$PATH"  # prefer system node over linuxbrew
  IMPORTANT: SSH login shells read ~/.bash_profile, not ~/.bashrc. Without both files,
  new SSH sessions will still pick up linuxbrew node v25.9.0. Each new SSH session
  opened during troubleshooting needs `source ~/.bash_profile` until fixed.

  Step 3: Verify in new shell:
    bash -i -c "which node && node --version"
    # Should show: /home/ubuntu/.local/bin/node / v24.15.0

Note: Do NOT remove the linuxbrew node binary — other linuxbrew packages depend on it
  (@salesforce CLI, playwright, speedtest-net in node_modules). Shadow it with PATH only.

Note: The TUI freeze leaves the terminal in raw/alternate-screen mode. If this happens,
  type `reset` or close and reopen the terminal session.

### Hermes cron job workdir set to path on wrong EC2 (silent failure)
Symptom: Hermes cron job runs, scheduler says "ok", but nothing executes.
  FileNotFoundError: /home/ubuntu/.openclaw/workspace
Cause: Hermes cron workdir points to a path on OpenClaw EC2, not Hermes EC2.
Fix: Clear workdir — a cron job that SSHs elsewhere doesn't need a local workdir.

### ServBay node taking PATH precedence over homebrew (Mac, May 2026)

Symptom: `openclaw --version` reports 2026.4.29 on Mac even after upgrading homebrew copy.
  `which openclaw` returns `/Applications/ServBay/package/node/current/bin/openclaw`.
  ServBay is a local web development environment that prepends its own node/bin to PATH
  via its shellenv init in ~/.zshrc or ~/.bash_profile.

Cause: ServBay installs its own Node + npm global packages, including openclaw, and puts
  its bin directory first in PATH. Homebrew's copy loses.

Diagnosis:
  bash -l -c 'which -a openclaw'
  (will show ServBay path first, then /opt/homebrew/bin/openclaw)

Fix:
  1. Uninstall openclaw from ServBay's npm global:
     /Applications/ServBay/package/node/current/bin/npm uninstall -g openclaw
     # verify: ls /Applications/ServBay/package/node/current/lib/node_modules/openclaw 2>/dev/null || echo removed

  2. Stop Mac LaunchAgents before upgrading homebrew copy:
     launchctl stop ai.openclaw.node
     launchctl stop ai.openclaw.node-host

  3. Install target version via homebrew npm:
     /opt/homebrew/bin/npm install -g openclaw@<version>

  4. Verify homebrew wins:
     bash -l -c 'which openclaw && openclaw --version'
     (should show /opt/homebrew/bin/openclaw)

  5. Restart LaunchAgents:
     launchctl start ai.openclaw.node
     launchctl start ai.openclaw.node-host

Key facts for Paul's Mac:
  - All LaunchAgents (ai.openclaw.node, ai.openclaw.node-host) are hardcoded to homebrew paths
    (/opt/homebrew/lib/node_modules/openclaw/dist/index.js and entry.js)
  - ServBay is completely irrelevant to LaunchAgents — removing from ServBay doesn't affect them
  - ~/.openclaw/ data dir is separate from installation — safe across any version change
  - Mac App (OpenClaw.app) ships its own bundled runtime — updates independently, not via npm

Note: OPENCLAW_SERVICE_VERSION in ai.openclaw.node.plist may still show old version after
  upgrade — this is cosmetic (an env var label only), does not affect functionality.

### Rocky Dashboard server.js calling openclaw status on every poll
Symptom: Child process pairs keep respawning, memory climbs toward 7GB+.
Cause: dashboard server.js calls `openclaw status` on every refresh poll.
Fix: Stop dashboard service, kill child pairs, patch server.js to use
  `systemctl --user is-active openclaw-gateway.service` instead.
Prevention: Never call openclaw CLI from a web server poll loop.

### Notion task cron jobs blocked by 2026.4.15 exec security
Symptom: Cron jobs spawn child processes that immediately fail and respawn, piling up.
  Log shows: exec preflight: complex interpreter invocation detected; refusing to run
Cause: 2026.4.15 blocks inline heredoc patterns (cat << PYEOF) and complex shell pipelines.
  Only direct file execution allowed: python3 file.py or node file.js
Fix: disable the affected cron jobs, clear stale runningAtMs from jobs.json, restart gateway.
  Then rewrite cron job messages to write Python to a .py file first, then execute it.
Jobs file: ~/.openclaw/cron/jobs.json (edit directly, gateway picks up on restart)

### Web UI access from remote devices (iPad, external browser)

The OpenClaw web UI canvas is at /__openclaw__/canvas/ — NOT at /openclaw or the gateway root.

Two access URLs for Paul's setup (as of 2026-05-02):
  https://rocky.onefellswoop.com.au/__openclaw__/canvas/   (via Caddy)
  https://rocky-on-ec2.tail9c7933.ts.net/__openclaw__/canvas/  (via Tailscale — requires Tailscale connected on device)

Common problems and fixes:

PROBLEM: rocky.onefellswoop.com.au/openclaw gives white page
  Cause: /openclaw* Caddy route proxies the gateway root — no web UI content there.
  Fix: Navigate to /__openclaw__/canvas/ path instead.
  Note: Caddy now has BOTH routes (/__openclaw__* and /openclaw*) — added 2026-05-02.

PROBLEM: Tailscale URL gives login screen but "origin not allowed" error on submit
  Cause: gateway.controlUi.allowedOrigins did not include the Tailscale hostname.
  Fix: Add to allowedOrigins in ~/.openclaw/openclaw.json:
    python3 -c "
    import json
    with open('/home/ubuntu/.openclaw/openclaw.json') as f: cfg = json.load(f)
    origins = cfg['gateway']['controlUi']['allowedOrigins']
    for o in ['https://rocky-on-ec2.tail9c7933.ts.net', 'https://rocky-on-ec2.tail9c7933.ts.net:443']:
        if o not in origins: origins.append(o)
    with open('/home/ubuntu/.openclaw/openclaw.json', 'w') as f: json.dump(cfg, f, indent=2)
    "
  Then restart gateway. Fix is persistent across restarts.

PROBLEM: Web UI loads, "hi" sent, chat bubble appears then disappears, no response after 3+ minutes
  Cause: Event loop blocked — almost always Telegram fetch-timeout starvation on 2026.4.29.
  See: "Telegram getMe health checks block event loop" section above.
  Diagnosis: Check gateway logs for repeated "[fetch-timeout] fetch timeout reached" for api.telegram.org.
  Fix: Disable Telegram channel, restart gateway.

Current allowedOrigins (Paul's setup, confirmed 2026-05-02):
  http://localhost:18789
  http://127.0.0.1:18789
  https://rocky.onefellswoop.com.au
  https://rocky.onefellswoop.com.au:443
  https://rocky-on-ec2.tail9c7933.ts.net
  https://rocky-on-ec2.tail9c7933.ts.net:443

### skills-remote probe timeout in logs (harmless)
Symptom: Logs repeatedly show "remote bin probe timed out (Paul's MacBook Pro)"
Cause: EC2 cannot SSH back to the Mac — expected, it can only go Mac -> EC2.
Action: None — harmless noise, not an error.

### TUI Fatal Deadlock in v2026.4.29 — "connecting | idle" freeze, code=1008 (GitHub #75844)
Symptom: TUI renders "connecting | idle", then "gateway disconnected: closed | idle".
  Gateway logs show "connect challenge timeout" code=1008 — the TUI never sends the auth frame
  back after the gateway issues its challenge. TUI is unresponsive to Ctrl-C (alternate-screen).
  Gateway itself is healthy. Web UI at http://127.0.0.1:18789/ works fine.
Cause: Confirmed regression in v2026.4.29 (GitHub issue #75844, open as of 2026-05-02).
  The TUI's JS event loop blocks before it can process the incoming WebSocket challenge frame.
  Not a gateway problem, not a token problem, not a sessions.json problem.
Workaround: Use the web UI via https://rocky.onefellswoop.com.au/openclaw while the regression
  is open. The gateway and all channels continue working normally.
Diagnostic confirmation: With explicit --token flag, TUI gets further ("agent main | session main
  | unknown | tokens ?") but still never fully connects — confirms the block is post-auth-frame,
  in the session/agent setup phase. This maps to #75656 (sync fs.readFileSync blocking event loop
  during session prep).
Note: The "connecting | idle" display is stale cached state rendered from disk by Ink UI — it
  does not mean the connection succeeded.

### Telegram getMe health checks block event loop / hang entire gateway (GitHub #75834, v2026.4.29)

Symptom: TUI hangs, web UI sends "hi" and gets no response (bubble disappears after 3+ minutes).
  Gateway logs show repeated fetch-timeout for Telegram getMe health checks, each taking
  13-56 seconds. Followed by eventLoopDelayP99Ms=56707 (56 seconds), eventLoopUtilization=1.
  models.list takes 194 seconds. sessions.list takes 11+ seconds.
  This LOOKS like a gateway or model problem but is actually Telegram blocking the event loop.

Cause: On 2026.4.29, Telegram's getMe health check probes do not respect the configured
  timeout on IPv4 fallback. Both [ariel] and [default] Telegram bot accounts each issue
  getMe checks every ~60s, each hanging for up to 56 seconds. Since Node.js is single-threaded,
  these blocking network calls starve the entire gateway event loop — no WebSocket handshakes
  can complete, no model calls can dispatch, web UI chat gets no response.

Fix (immediate — disables Telegram to restore gateway):
  python3 -c "
  import json
  with open('/home/ubuntu/.openclaw/openclaw.json', 'r') as f:
      cfg = json.load(f)
  cfg['channels']['telegram']['enabled'] = False
  cfg['channels']['telegram']['accounts']['ariel']['enabled'] = False
  with open('/home/ubuntu/.openclaw/openclaw.json', 'w') as f:
      json.dump(cfg, f, indent=2)
  print('Telegram disabled')
  "
  systemctl --user restart openclaw-gateway
  # Gateway startup drops from 24s to 17s. No fetch-timeout noise. Event loop clean.

Re-enable Telegram once 2026.4.29 is patched (check GitHub #75834 for fix status).

Note: The symptom looks EXACTLY like the zombie-TUI event loop starvation issue — same
  eventLoopDelayP99 spikes, same handshake timeouts, same web UI unresponsiveness. Distinguish
  them by checking WHAT is flooding the logs: zombie-TUI → many WS connection attempts;
  Telegram fetch-timeout → repeated "[fetch-timeout] fetch timeout reached" for api.telegram.org.

Reference: GitHub openclaw/openclaw #75834 (open as of 2026-05-02).

### Telegram channel fails to load: "Cannot find module 'grammy'" (GitHub #75818, v2026.4.29)

Symptom: Gateway starts but Telegram channel never comes up. Doctor or gateway logs show:
  "[channels] failed to load bundled channel telegram: Cannot find module 'grammy'"
  Also: "[channels] failed to load bundled channel telegram: Cannot find module '../dist/babel.cjs'"
  Telegram fetch timeouts at startup are a secondary symptom — Telegram never actually loads.

Cause: On a fresh install or upgrade to 2026.4.29, the telegram (and likely discord/nostr/slack)
  bundled extension cannot resolve its own runtime dependencies on first run. The packaging
  step doesn't stage grammy and related deps in the right location.

Fix (confirmed on Paul's EC2, 2026-05-02):
  openclaw doctor --fix
  systemctl --user restart openclaw-gateway
  # Doctor stages all 49 bundled runtime deps including grammy (78ms install).
  # On next restart: "[plugins] telegram installed bundled runtime deps in 78ms"
  # No more "Cannot find module" errors. Telegram loads cleanly.

Note: The "[telegram] fetch fallback: enabling sticky IPv4-only dispatcher" line still
  appears after the fix — that's the separate undici IPv6 issue (#75539), not grammy.
  Once you see that line (and NOT a "Cannot find module" error), Telegram has loaded.

Note: doctor --fix also stages deps for memory-core, runway, senseaudio, tts-local-cli,
  voyage, web-readability, xai and others. Safe to run at any time.

Reference: references/github-issues-v2026.4.29.md — #75818 section.


### Undici IPv6-first blocks event loop on IPv4-only EC2 (GitHub #75539, v2026.4.29)
Symptom: Gateway logs show "fetch timeout reached; aborting operation" for Telegram on startup.
  Followed by "enabling sticky IPv4-only dispatcher". Event loop spikes of 12-31 seconds
  immediately after boot while Telegram tries IPv6 connections that timeout before IPv4 fallback.
  TUI connection attempts during this startup window all fail (15s handshake budget exceeded).
Cause: undici v8.1.0 (bundled in plugin-runtime-deps in v2026.4.29) uses happy-eyeballs IPv6-first.
  EC2 has no global IPv6 route. api.telegram.org resolves both A and AAAA — undici tries AAAA
  first, waits for timeout, then falls back. Each attempt blocks the event loop.
  The gateway's automatic "sticky IPv4-only" fallback eventually kicks in, but the startup window
  is hazardous for TUI connections.
Partial workaround (add to systemd override.conf):
  Environment="NODE_OPTIONS=--dns-result-order=ipv4first"
Note: This workaround is PARTIAL — the bundled undici ignores Node.js DNS settings and uses its
  own resolver. The gateway's auto-fallback to IPv4 is the actual mitigant. Once it triggers
  (visible in logs as "enabling sticky IPv4-only dispatcher"), subsequent Telegram calls are fast.
  Wait ~60s after gateway start before attempting TUI connections.
Tracking: GitHub openclaw/openclaw #75539 (open), #75759 (idle CPU + Telegram timeouts, same root).

### Synchronous session transcript reads block event loop during agent prep (GitHub #75656)
Symptom: Gateway logs show agent prep totalMs > 20000ms. Common breakdowns:
  core-plugin-tools: 40000ms+, system-prompt: 30000ms+, stream-setup: 30000ms+
  During prep, no WebSocket handshakes can complete — TUI gets code=1008 challenge timeout.
  Also: sessions.list calls taking 16-77 seconds, stacking if Mac app polls during this window.
Cause: session-utils uses synchronous fs.readFileSync() during agent prompt-token estimation and
  session event sequencing. With accumulated session transcripts, blocks the event loop for
  tens of seconds. Separate root cause from the WAL bloat issue — this is transcript reads, not
  SQLite. Confirmed present in v2026.4.29.
Mitigation: Keep sessions.json trimmed (see sessions.json bloat section). Disconnect Mac app
  while troubleshooting TUI (app's sessions.list polling compounds the blocking).
Tracking: GitHub openclaw/openclaw #75656 (open as of 2026-05-02).

### npm install gets OOM-killed during openclaw update
Symptom: npm install of openclaw gets SIGKILL mid-process.
Cause: Gateway + other services consuming most RAM; npm install needs ~1.5GB for native
  compile (opus audio library). OOM killer fires.
Fix: Stop gateway before upgrading, install, then restart gateway.

### Gateway startup dramatically faster in 2026.4.29
Previously startup on this EC2 took 135-145 seconds. After upgrading to 2026.4.29, startup
takes ~37 seconds. Do NOT wait 2-3 minutes — port 18789 should be up in under a minute.
Timeline: config (~1s) -> auth (~0.1s) -> plugins staged/installed (browser+telegram, ~30s)
  -> HTTP server listening -> channels start -> gateway ready (~37s total).
Root cause of improvement: plugin runtime-deps no longer relaunching on startup.
Note: An event loop spike is normal immediately post-startup (P99 ~3.4 seconds, max ~3.4 seconds
  in the first 30-second window) as plugins initialise. It settles fast — by the next
  30-second diagnostic window it was already down to P99 21ms. Do not treat this spike
  as a problem; wait one diagnostic cycle before worrying.
The "Connectivity probe: ok" + "Read probe: failed - timeout" shape from `openclaw gateway probe`
  is normal on this EC2 — the 3000ms probe budget is too short for the RPC round-trip.
  Use `timeout 30 openclaw gateway probe` to get a meaningful result.

### Startup optimisations for low-resource hosts (EC2 t-class etc.)
Add to the systemd service override.conf:
  Environment="NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache"
  Environment="OPENCLAW_NO_RESPAWN=1"
Create the cache dir first: mkdir -p /var/tmp/openclaw-compile-cache


## Ariel — Second OpenClaw Agent (workspace-ariel)

A second OpenClaw agent named Ariel runs on the same EC2 (52.63.13.24) with a separate
workspace. She has a different structure from the main OpenClaw agent — no sessions.json,
no WAL files, no cron jobs of her own.

Workspace path: ~/.openclaw/workspace-ariel/
Internal OpenClaw state: ~/.openclaw/workspace-ariel/.openclaw/workspace-state.json

Key files to monitor:
  WORKSTATE.md              — Ariel's live hot checkpoint; updated during active work
  memory/.dreams/short-term-recall.json — her primary memory store (watch for bloat)
  memory/.dreams/events.jsonl           — memory event log
  .openclaw/workspace-state.json        — workspace bootstrap metadata
  projects/                             — per-project files
  state/                                — 1password cache and other live state

Health signals:
  WORKSTATE.md last-modified > 7 days   → escalate (Ariel may be stuck)
  short-term-recall.json > 100KB        → flag in report (she manages her own memory)
  short-term-recall.json > 200KB        → escalate (memory bloat risk)
  workspace-state.json missing/corrupt  → escalate
  workspace directory missing entirely  → escalate

How to check:
  ssh -i ~/.ssh/openclaw-ec2.pem ubuntu@52.63.13.24 '
    ls -la ~/.openclaw/workspace-ariel/ 2>/dev/null || echo workspace missing
    cat ~/.openclaw/workspace-ariel/.openclaw/workspace-state.json 2>/dev/null
    ls -lah ~/.openclaw/workspace-ariel/memory/.dreams/short-term-recall.json 2>/dev/null
    ls -lah ~/.openclaw/workspace-ariel/memory/.dreams/events.jsonl 2>/dev/null
    stat -c "%y" ~/.openclaw/workspace-ariel/WORKSTATE.md 2>/dev/null
  '

Key differences from main agent:
  - No sessions.json (does not accumulate run histories)
  - No WAL files (no SQLite task/flow databases)
  - No cron jobs of her own
  - Memory managed as flat JSON files, not a relational store
  - Workspace sync via workspace-sync.sh + LaunchAgent plist on Mac side

Current active work (as of 2026-04-30): Bellevue Geelong website — Hero Banner component,
  theme CSS in bellevue-full, flexible content components pass underway.


## Version Stability (May 2026)

2026.4.29 has severe regressions (73s agent prep, TUI deadlock, silent no-reply on OpenAI).
Recommended stable: 2026.4.23
Primary model must be Anthropic — OpenAI transport silently drops replies on 4.29.
Full details + downgrade procedure: references/version-stability-may2026.md

Key pitfalls for version evaluation:
- Reddit is BLOCKED from EC2/datacenter IPs — GitHub issues API is the only live community signal
- op (1Password CLI) is NOT installed on the Hermes EC2 — SSH to OpenClaw EC2 (52.63.13.24)
  to retrieve 1Password secrets when running from Hermes sessions or cron jobs
- sendMail via MS365 Graph API returns HTTP 202 with EMPTY body on success — empty response
  is correct, not an error

## Version Review Records

Past version upgrade reviews are in references/:
- references/version-review-2026.4.21-to-2026.4.29.md — breaking changes, changelog technique, Paul's config state at time of review
- references/version-stability-notes.md — per-version stability assessment, known-broken versions, key GitHub issues, Mac install details
- references/version-stability-may2026.md — May 2026 stability table (current recommended: 2026.4.23)
- references/version-evaluation-workflow.md — full evaluation procedure: curl commands, scoring rubric, automated cron job details

## Reference Files (in this skill)

- templates/ec2-upgrade-procedure.md — full step-by-step upgrade runbook for this EC2
- references/changelog-2026.4.29-notes.md — breaking changes and relevant fixes for the 2026.4.29 upgrade
- references/version-upgrade-notes.md — per-version upgrade notes from real sessions
- references/github-issues-v2026.4.29.md — known open GitHub issues in v2026.4.29 (TUI deadlock #75844, undici IPv6 #75539, sync reads #75656, idle CPU #75759)

## Rocky's Access Path to Paul's Mac (confirmed May 2026)

Rocky (Hermes EC2 172.31.30.215) can reach Paul's Mac via a two-hop path:
  Rocky → OpenClaw EC2 (172.31.20.31 via SSH key ~/.ssh/openclaw-ec2.pem)
       → Mac (100.87.144.6 via Tailscale, user paulm)

Commands:
  # From Rocky:
  ssh -i ~/.ssh/openclaw-ec2.pem ubuntu@172.31.20.31 'ssh paulm@100.87.144.6 "<command>"'

Confirmed working after EC2 reboot (May 2026). Tailscale on EC2 maintains connectivity to Mac
even when the Mac SSH tunnel (ai.openclaw.gateway-tunnel LaunchAgent) has dropped and reconnected.

The reverse tunnel (ai.openclaw.gateway-tunnel) is a LaunchAgent on the Mac that forwards
Mac's localhost:18789 to EC2 port 18789. It is required for OpenClaw functions — do NOT disable it.
It will show exit code 255 in launchctl after a dropped connection — this is normal KeepAlive
reconnect behaviour, not a failure. Verify it's actually working with:
  ssh paulm@100.87.144.6 "curl -s -o /dev/null -w '%{http_code}' --max-time 5 http://127.0.0.1:18789/"
  # Should return 200 if tunnel is healthy

Mac openclaw binary location: /opt/homebrew/bin/openclaw
Mac openclaw config: /Users/paulm/.openclaw/openclaw.json
Note: `node` is NOT in PATH for non-login SSH sessions to Mac. Use full paths where needed.

## Paul's Environment — Quick Reference (May 2026)

ALWAYS read ENVIRONMENT.md first for full architecture:
  ssh -i ~/.ssh/openclaw-ec2.pem ubuntu@172.31.20.31 "cat ~/.openclaw/workspace/ENVIRONMENT.md"

Key facts:
- EC2 active install: /usr/lib/node_modules/openclaw (sudo npm install -g)
  NOT linuxbrew (linuxbrew copy was removed May 2026)
- Gateway service: ~/.config/systemd/user/openclaw-gateway.service
- Node PATH fix: ~/.local/bin/node symlink points to /usr/bin/node v24.15.0
  Fix is in BOTH ~/.bashrc AND ~/.bash_profile (login shells need .bash_profile)
  New SSH sessions are login shells — always check `which node` before running TUI
- Mac connects to EC2 via: wss://rocky.onefellswoop.com.au/openclaw (Caddy proxy)
- Mac TUI gateway.remote.url must be: wss://rocky.onefellswoop.com.au/openclaw
- MagicDNS is OFF on Paul's Mac — Tailscale hostnames do not resolve
- Mac node host LaunchAgent: ai.openclaw.node.plist (connects to EC2 gateway)
- Mac app (menubar): separate from node host, connects independently
- Caddy Caddyfile: /etc/caddy/Caddyfile — routes /openclaw* to 127.0.0.1:18789
- WhatsApp warnings in Mac openclaw config output: harmless, disabled plugin noise
- Current version (May 2026): 2026.4.29 on EC2, 2026.4.21 on Mac CLI — TARGET DOWNGRADE TO 2026.4.23
- Rocky MUST NOT run openclaw tui via SSH — TERM=dumb, always fails, leaves zombie processes at 90% CPU
- Web UI workaround: http://localhost:18789 via Mac tunnel (token required on first load)
- Gateway registers itself with Tailscale serve at startup (sudo tailscale serve --bg --yes 18789)
  providing a second access path: wss://rocky-on-ec2.tail9c7933.ts.net (alongside Caddy)
- OFS-Onyx-MA uvicorn app runs on EC2 port 3760 (python3.14, workspace/OFS-Onyx-MA/Milestone1)
- Mac remote access: from OpenClaw EC2, ssh paulm@100.87.144.6 (Tailscale, no key needed)
- Mac dashboard processes (normal, not a problem): Rocky_dashboard/server.js, Studio Agent/dashboard/server.js
- Mac tunnel LaunchAgent: ai.openclaw.gateway-tunnel (KeepAlive, INTENTIONAL — do not disable)


## Key Docs Pages

- Getting started: https://docs.openclaw.ai/start/getting-started
- Install: https://docs.openclaw.ai/install
- CLI reference: https://docs.openclaw.ai/cli
- Gateway runbook: https://docs.openclaw.ai/gateway
- Gateway troubleshooting: https://docs.openclaw.ai/gateway/troubleshooting
- Channels: https://docs.openclaw.ai/channels
- Tools & plugins: https://docs.openclaw.ai/tools
- Providers: https://docs.openclaw.ai/providers
- General troubleshooting: https://docs.openclaw.ai/help/troubleshooting
- Security: https://trust.openclaw.ai
- VPS hosting: https://docs.openclaw.ai/vps
