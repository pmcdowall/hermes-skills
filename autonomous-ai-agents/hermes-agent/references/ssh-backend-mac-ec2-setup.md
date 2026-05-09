# Hermes SSH Backend: Mac (brain) → EC2 (execution)

Verified setup pattern: Hermes on macOS with terminal/file tools running
over SSH on an EC2 instance. Tailscale provides the network path.

## What you end up with

- Mac Hermes session — chat, memory, skills, macos-computer-use all local
- Terminal commands — transparently run on EC2 via SSH
- File tools (read_file, write_file, patch, search_files) — run on EC2
- Memory and sessions — stored on the Mac (separate from any EC2 instance)
- computer-use — drives Mac desktop apps in the background (local only)

## Prerequisites

- Hermes already running on EC2
- Tailscale installed and connected on both Mac and EC2
- SSH key that grants access to EC2 (the key the Mac already uses to SSH in)

## Step 1 — Install Hermes on the Mac

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
hermes setup   # walk through model + API key config
```

Use the same API key/provider as the EC2 instance, or a different one —
they're fully independent.

## Step 2 — Configure the SSH terminal backend

```bash
hermes config set terminal.backend ssh
```

Then edit `~/.hermes/.env` on the Mac and add:

```
TERMINAL_SSH_HOST=100.87.144.6        # Tailscale address — works from anywhere
TERMINAL_SSH_USER=ubuntu
TERMINAL_SSH_KEY=/Users/paulm/.ssh/your-ec2-key.pem   # adjust path
TERMINAL_SSH_PORT=22
```

Use the Tailscale address, not the EC2 internal IP (172.31.x.x) — the
Tailscale address works whether you're at home, on mobile, or anywhere else.

## Step 3 — Enable computer-use on the Mac

```bash
hermes tools
```

Find "Computer Use" in the list and enable it. This installs cua-driver.
macOS will prompt for two permissions — grant both:
- Accessibility (System Settings → Privacy & Security → Accessibility)
- Screen Recording (System Settings → Privacy & Security → Screen Recording)

## Step 4 — Verify

```bash
hermes doctor
hermes config   # should show: Backend: ssh, SSH host: 100.87.144.6
```

Start a session and run a terminal command — it executes on EC2. Run a
computer_use capture — it shows the Mac screen.

## Phase 2 — Shared skills via git tap

Both Hermes instances pull skills from a shared private GitHub repo, so a
skill built once is available everywhere.

### Setup (do this once)

**On EC2** — initialise the skills directory as a git repo and push:

```bash
cd ~/.hermes/skills
git init && git branch -m main
git remote add origin https://github.com/pmcdowall/hermes-skills.git
git config user.email "paulm@onefellswoop.com.au"
git config user.name "Paul McDowall"
git add -A && git commit -m "Initial skills snapshot"
# Push requires a GitHub PAT — gh CLI is NOT installed on EC2
# Set git credential: git config credential.helper store
# Then: git push -u origin main  (enter username + PAT when prompted)
```

**Pitfall: `gh` CLI is not installed on Hermes EC2.**
The `gh` CLI is only on the Mac. To push from EC2 you need a GitHub PAT
configured via `git config credential.helper store` or passed inline.
Alternative: do the initial push from the Mac instead —
  1. Clone the repo on Mac
  2. Copy skills from EC2 via `rsync -a ubuntu@100.113.102.39:~/.hermes/skills/ ~/hermes-skills/`
  3. Commit and push from Mac
  4. Add tap on both machines

**On both Mac and EC2** — add the tap:

```bash
hermes skills tap add https://github.com/pmcdowall/hermes-skills.git
```

Skills from the tap appear alongside local skills automatically.
Run `hermes skills check && hermes skills update` to pull latest from the repo.

### Keeping in sync

After creating or updating a skill on either machine:

```bash
cd ~/.hermes/skills
git add -A && git commit -m "Update: skill-name"
git push
```

On the other machine:
```bash
cd ~/.hermes/skills && git pull
```

Or run `hermes skills update` if the tap handles pull automatically.

---

## Notes on session isolation

The two Hermes instances (Mac + EC2) have separate memory and sessions.
The Mac Rocky starts fresh — no knowledge of EC2 Rocky's history.

To share context between them, options include:
- The `shared-agent-context` skill (reads/writes shared files both can see)
- Pointing both at the same Tailscale-accessible file share
- Explicitly copying ~/.hermes/config.yaml across (for config parity)

## SSH connection details (from tools/environments/ssh.py)

- ControlMaster=auto, ControlPersist=300 — one persistent connection,
  reused for all tool calls in the session
- StrictHostKeyChecking=accept-new — first connection auto-accepts host key
- Socket path: sha256(user@host:port)[:16].sock in /tmp/hermes-ssh/
  (hashed to stay under macOS 104-byte sun_path limit)
- On session start: syncs ~/.hermes/ from local to remote so the remote
  has current skills and config
- ConnectTimeout=10, BatchMode=yes (no interactive password prompts)
