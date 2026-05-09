# OpenClaw Version Upgrade Notes

Notes accumulated during real upgrade sessions on the EC2. Supplement to the official changelog.

---

## 2026.4.21 → 2026.4.29 (upgraded May 2026)

### Breaking change: tools.exec / tools.fs no longer widen restrictive profiles

Config key: `tools.exec`, `tools.fs`
Impact: Only affects installs using `tools.profile = "messaging"` or `"minimal"`.
This EC2 uses the default "full" profile (no profile key set) — NOT affected.
If you ever add a restrictive profile in future, explicit `alsoAllow` entries are required
to re-enable exec and fs tools.

### Gateway systemd exit code 78 for EADDRINUSE / lock conflicts

New: gateway now exits with sysexits 78 on port conflict or lock collision, enabling
`RestartPreventExitStatus=78` in the service file to stop restart loops.
Practical benefit: reduces the systemctl restart mid-boot kill problem.

### Subagent orphan recovery (sessions.json surgery less needed)

New: gateway bounds orphan recovery with persisted recovery attempts and tombstone.
`openclaw doctor` can now reconcile wedged sessions automatically.
Implication: manual sessions.json surgery for dead subagents is less necessary post-4.29.

### Plugin runtime-deps improvements

Multiple fixes for plugin staging, stale symlink handling, and incomplete install repair.
Practical: stop gateway before updating, then allow `openclaw doctor --repair` to fix
any plugin install issues on first post-update start.

### Startup time improved dramatically

On this EC2: startup went from 135-145 seconds (v2026.4.15) to ~5 seconds (v2026.4.29).
The startup model warmup still times out (5000ms grace), but gateway reaches "ready" fast.
Plugin runtime-dep staging now completes in 15-20ms per plugin (was much slower).

### Stale models config suppression (Codex / gpt-5.4-mini)

If `openclaw doctor --fix` previously wrote openai-codex/gpt-5.4-mini model entries to
your config, 2026.4.29 silently suppresses them. No action needed, but if model selection
behaves unexpectedly after upgrade, check your models config for stale entries.

### TUI hang on Node 25 (NOT documented officially)

See "Linuxbrew node version conflict" in the main skill. This is an undocumented
incompatibility between the 2026.4.29 TUI and Node 25.x. Workaround: ensure the
interactive shell uses Node 24 via ~/.local/bin/node symlink.

### Changelog location

GitHub raw: https://raw.githubusercontent.com/openclaw/openclaw/main/CHANGELOG.md
Use `curl -fsSL <url> | awk '/^## <version>/,/^## <previous>/'` to extract a specific
version section for review before upgrading.

---

## Version naming convention

OpenClaw uses date-based versions: YYYY.M.DD (e.g. 2026.4.29 = April 29, 2026).
There is no UPGRADING.md or MIGRATION.md in the repo — all changes are in CHANGELOG.md.
The `main` branch package.json always reflects the latest unreleased version.
To see the changelog for a specific published version, check the relevant git tag or npm.

---

## Node.js requirements (as of 2026.4.29)

Minimum: Node >= 22.14.0
Recommended: Node 24.x
This EC2 runs: v24.15.0 (system npm at /usr/bin/node)
Linuxbrew node: v25.9.0 (do NOT use for openclaw — causes TUI hang)
