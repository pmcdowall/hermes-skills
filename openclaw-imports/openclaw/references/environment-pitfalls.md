# OpenClaw — Environment & Tooling Pitfalls

Last updated: 2026-05-03

## Critical: Hermes workdir breakage

Hermes sets `/home/ubuntu/.openclaw/workspace` as the default terminal working directory.
If that directory is absent (e.g. after a botched upgrade, uninstall, or OpenClaw workspace
reset), **ALL terminal, file, and search tools fail** with:

```
FileNotFoundError: [Errno 2] No such file or directory: '/home/ubuntu/.openclaw/workspace'
```

This is a silent, total failure — commands return empty output and exit code -1. It's easy to
mistake for a network issue or permission problem.

**Fix:** `mkdir -p /home/ubuntu/.openclaw/workspace`

But since the terminal is broken, you can't run this via normal tooling. Options:
1. SSH to EC2 directly from Mac via Tailscale: `ssh paulm@100.87.144.6` then `ssh ubuntu@172.31.20.31`
2. Use the EC2 AWS console if SSH is also broken
3. If the Hermes gateway is still running, a cron job or external trigger can run the mkdir

After creating the directory, Hermes tool calls should resume normally without restart.

## Subagent browser/web toolsets do not work on EC2

Chrome is not installed in subagent containers on EC2. Delegating tasks with
`toolsets: ["browser"]` or `toolsets: ["browser", "web"]` will always fail:

```
Auto-launch failed: Chrome not found. Checked:
  - agent-browser cache: /home/ubuntu/.agent-browser/browsers
  - System Chrome installations
  - Puppeteer browser cache
  - Playwright browser cache
Run `agent-browser install` to download Chrome, or use --executable-path.
```

The `web` toolset alone also returned no results in practice (subagent reported no tools
available). Do not delegate web/GitHub research to subagents in this environment.

**Workaround:** Run curl directly from the main agent terminal using the GitHub REST API:

```bash
# Latest releases
curl -s 'https://api.github.com/repos/openclaw/openclaw/releases?per_page=8'

# Open issues
curl -s 'https://api.github.com/repos/openclaw/openclaw/issues?state=open&per_page=20'

# Specific issue
curl -s 'https://api.github.com/repos/openclaw/openclaw/issues/75999'

# Raw CHANGELOG from main branch
curl -fsSL https://raw.githubusercontent.com/openclaw/openclaw/main/CHANGELOG.md | head -100

# npm registry info
npm view openclaw versions --json | python3 -c "import json,sys; v=json.load(sys.stdin); print('\n'.join(v[-10:]))"
```

## Terminal not working at all?

If all terminal commands return empty output and exit code -1:
1. First suspect: workdir doesn't exist (see above)
2. Second suspect: OpenClaw gateway process is consuming all CPU/memory
3. Check: try `terminal("echo hello", workdir="/tmp")` — if that also fails, it's the workdir issue

## Upgrading OpenClaw — pre-flight checklist

Before upgrading from 4.23 to any new version:

1. Check CHANGELOG for: prep-stage changes, dispatch flow changes, plugin runtime-deps changes
2. Verify Node.js requirement (check `engines.node` in package.json on the new tag)
3. Check if sessions.json or SQLite schema changed (harder rollback if yes)
4. Confirm the known broken issues are actually closed with the release tag (not just fixed on main)
5. Check GitHub issues for new regressions opened in the 7 days after release
6. Check Reddit r/openclaw and the Discord for community reports
7. Take a backup: `cp -r ~/.openclaw ~/.openclaw.bak-$(date +%Y%m%d)` before upgrading
