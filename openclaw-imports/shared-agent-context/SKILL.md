---
name: shared-agent-context
description: Read and update the shared context files between Rocky (Hermes) and OpenClaw. Load this skill whenever picking up work that OpenClaw may have touched, or before any EC2 infrastructure work. Also enforces the update discipline — must write back to shared files after significant shared work.
triggers:
  - shared context
  - openclaw work
  - EC2 infrastructure
  - what did openclaw do
  - picking up where openclaw left off
  - shared memory
  - agent context
confirmed: true
---

# Shared Agent Context — Rocky (Hermes)

## What This Is

Rocky (Hermes) and Rocky (OpenClaw) run on SEPARATE EC2 instances and work on
overlapping infrastructure and projects for Paul. This skill governs how we stay in sync.

Hermes EC2:   172.31.30.215 (this machine)
OpenClaw EC2: 172.31.20.31  (OpenClaw's machine, also accessible via 52.63.13.24)

Two agents run on the OpenClaw EC2:
  Main OpenClaw agent — ~/.openclaw/agents/main/ (sessions.json, WAL files, cron)
  Ariel              — ~/.openclaw/workspace-ariel/ (workspace-based, no sessions.json/WAL)

The canonical ~/.shared/ repo lives on OpenClaw's EC2 at 172.31.20.31.
Hermes accesses it via SSH using key: ~/.ssh/openclaw-ec2.pem
User: ubuntu

Hermes does NOT have a local ~/.shared/ git repo that matters.
The source of truth is always on OpenClaw's EC2.


## Files

~/.shared/context/common_context.md
  -- Who Paul is, OFS, key conventions, active projects.
  -- The primary briefing document. Stable facts, updated in place.

~/.shared/context/ec2-state.md
  -- Live EC2 infrastructure state. Updated after any infrastructure work.
  -- Critical for avoiding "I didn't know the WAL was already broken" situations.

~/.shared/skills/ofs-conventions.md
  -- OFS naming, tools, workflows, preferences.

~/.shared/skills/ec2-operations.md
  -- Safe procedures for EC2 work: start/stop sequences, WAL checkpoint, trim.


## When to Load Shared Context

Load at the start of any session where you:
- Are picking up work OpenClaw started or last touched
- Are about to do EC2 infrastructure work (gateway, OpenClaw, WAL, tokens)
- Are working on anything OFS-related that OpenClaw might also touch
- Paul mentions something OpenClaw told him

In practice: if there's any doubt, read it. The files are small.


## Files

All files live on OpenClaw's EC2 at ubuntu@172.31.20.31:~/.shared/

~/.shared/context/common_context.md   -- who Paul is, OFS, both agents, active projects
~/.shared/context/ec2-state.md        -- live EC2 infra state, updated after any infra work
~/.shared/skills/ofs-conventions.md   -- OFS naming, tools, workflows
~/.shared/skills/ec2-operations.md    -- safe EC2 procedures


## How to Read

SSH-cat the files from OpenClaw's EC2:

  terminal("ssh -i ~/.ssh/openclaw-ec2.pem ubuntu@172.31.20.31 'cat ~/.shared/context/common_context.md'")
  terminal("ssh -i ~/.ssh/openclaw-ec2.pem ubuntu@172.31.20.31 'cat ~/.shared/context/ec2-state.md'")

For EC2 or OFS work also read:

  terminal("ssh -i ~/.ssh/openclaw-ec2.pem ubuntu@172.31.20.31 'cat ~/.shared/skills/ec2-operations.md'")
  terminal("ssh -i ~/.ssh/openclaw-ec2.pem ubuntu@172.31.20.31 'cat ~/.shared/skills/ofs-conventions.md'")


## When to Update — THE DISCIPLINE RULE

After any session involving:
- EC2 infrastructure changes (gateway restart, WAL fix, token change, sessions trim)
- New projects or significant decisions Paul has made
- Anything OpenClaw should know before picking up next
- New OFS conventions or tool access patterns discovered

You MUST update the relevant shared file before closing the session.
This is not optional. A stale shared context is worse than no shared context
because it creates false confidence.


## How to Update

Step 1 — SCP the updated file to OpenClaw's EC2:

  terminal("scp -i ~/.ssh/openclaw-ec2.pem /tmp/updated_file.md ubuntu@172.31.20.31:~/.shared/context/common_context.md")

  Or for small edits, write via SSH heredoc:
  Use terminal() to construct the new content, write to a local temp file, then scp it.

Step 2 — Commit on OpenClaw's EC2 via SSH:

  terminal("ssh -i ~/.ssh/openclaw-ec2.pem ubuntu@172.31.20.31 \"cd ~/.shared && git add -A && git commit -m 'Rocky (Hermes): <brief description>'\"")

MANDATORY VERIFICATION — do not report success until this passes:

  terminal("ssh -i ~/.ssh/openclaw-ec2.pem ubuntu@172.31.20.31 'cd ~/.shared && git log --oneline | head -5'")

You must see your commit in the output before reporting done.
Do not tell Paul the work is done if you cannot show the commit hash.


## Pitfalls

- Do NOT append to common_context.md like a log. It's a briefing doc. Update in place.
- Do NOT skip the git commit. Version history is the safety net.
- Do NOT update ec2-state.md with guesses. Only write what you actually observed.
- If OpenClaw's TODO marker is still in ec2-state.md, leave it — don't remove it
  until OpenClaw has actually filled in current state.
