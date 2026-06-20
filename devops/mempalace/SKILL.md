---
name: mempalace
description: "MemPalace reference library: query, add, and manage the palace. Use when searching curated knowledge, adding new learnings, or managing palace structure."
version: 1.0.0
author: Rocky (Hermes)
platforms: [linux]
metadata:
  hermes:
    tags: [memory, knowledge, MCP, reference]
---

# MemPalace — Rocky's Reference Library

MemPalace is a local-first semantic knowledge store running on the Hermes EC2. It complements Honcho (who Paul is) and skills (how to do things) with curated reference knowledge that's too rich for a skill but too buried in sessions to recall cleanly.

## When to Use

- Paul says "add that to MemPalace" → file the key learning as a drawer
- Searching for a specific procedure, fact, or gotcha across projects
- Starting work on OFS Onyx MA → search ofs-onyx-ma wing first
- Checking infrastructure config → search hermes-infra wing
- After a session with significant learning → add drawers before finishing

## Installation Details (Hermes EC2)

- **Version:** 3.4.1 (updated 2026-06-20 from 3.4.0)
- **Install:** `/home/ubuntu/.hermes/hermes-agent/.venv` (Python 3.11; uv-managed — no `pip` binary, use `python -m pip`)
- **Palace path:** `/home/ubuntu/.mempalace/palace`
- **Global config:** `/home/ubuntu/.mempalace/config.json`
- **Embedding model:** minilm (English, all-MiniLM-L6-v2, ~79MB cached at `~/.cache/chroma/onnx_models/`). Identity recorded via `mempalace palace set-embedder --model minilm`.
- **MCP config:** `~/.hermes/config.yaml` under `mcp_servers.mempalace`
- **MCP command:** `/home/ubuntu/.hermes/hermes-agent/.venv/bin/python -m mempalace.mcp_server --palace /home/ubuntu/.mempalace/palace`
- **Tools registered as:** `mcp_mempalace_*`. MCP server is a subprocess of the Hermes gateway — restart `hermes-gateway.service` to respawn it on a new version.

## Update Procedure (verified 2026-06-20)

1. **Back up first (hard rule):** `tar czf ~/.mempalace-backup-$(date +%Y%m%d-%H%M%S).tar.gz -C ~/.mempalace .`
2. **Check latest:** `python -m pip index versions mempalace` (or `pip install 'mempalace=='` to list).
3. **Read the changelog** at `https://raw.githubusercontent.com/MemPalace/mempalace/develop/CHANGELOG.md` — watch for ChromaDB version bumps (need `mempalace migrate`) or embedder changes (need `repair rebuild-index`). 3.4.x has neither.
4. **Upgrade:** `<venv>/bin/python -m pip install --upgrade 'mempalace==X.Y.Z'`
5. **Restart** `hermes-gateway.service` (+ `hermes-gateway-sterling.service`) to respawn the MCP subprocess.
6. **Verify:** `mempalace --palace <p> status` (drawer count unchanged), `mempalace --palace <p> repair-status` (sqlite vs HNSW divergence=0, status OK).

**3.4.1 notes:** backup-retention pruning (`max_backups` default 10, env `MEMPALACE_MAX_BACKUPS`) — prevents stale migrate/repair backups silently filling disk. embeddinggemma OOM fix (n/a on minilm). Cursor/Antigravity IDE plugins (n/a — MCP-server-only here). No ChromaDB change, no re-embed.

## Palace Structure

```
hermes-infra/
  ec2-setup      — server identity, access, gateway service
  honcho         — Honcho setup, tools, config, peer-keying fix
  openclaw       — gateway arch, OOM incident+fix, oauth2 hardening, upgrade, WAL
  gateway        — Hermes gateway notes (Honcho latency fix etc.)
  monitoring     — external Telegram watchdog, hermes-send alert pattern
  mac-infra      — Mac devices, OpenClaw on Mac, HomeBridge, Tailscale
  mempalace      — MemPalace own install/config reference
  synology       — Synology NAS notes

ofs-onyx-ma/
  pipeline       — MA system overview, tech stack, deployment
  abs-geography  — SA2, ASGS, ABS API, concordance tables
  fastapi        — FastAPI + Jinja2 patterns, async pitfalls
  known-issues   — bugs, gotchas, ASGS mismatch, data suppression
  integrations   — external integration notes
```

## Key MCP Tools

| Tool | Use |
|------|-----|
| `mcp_mempalace_search` | Semantic search — primary retrieval tool |
| `mcp_mempalace_add_drawer` | File knowledge into a wing/room |
| `mcp_mempalace_get_taxonomy` | See full wing/room/count tree |
| `mcp_mempalace_list_wings` | List wings with drawer counts |
| `mcp_mempalace_list_rooms` | List rooms in a wing |
| `mcp_mempalace_get_drawer` | Fetch full content of one drawer |
| `mcp_mempalace_update_drawer` | Update existing drawer |
| `mcp_mempalace_kg_add` | Add entity relationship to knowledge graph |
| `mcp_mempalace_kg_query` | Query entity relationships |
| `mcp_mempalace_traverse` | Walk graph across wings |

**Note:** MCP tools only available after Hermes restart. Until restart, use the Python API directly (see Manual Operations below).

## Search Tips

```python
# Wing-scoped search (fastest, most precise)
mcp_mempalace_search(query="WAL checkpoint procedure", wing="hermes-infra")

# Cross-palace search
mcp_mempalace_search(query="SA2 ASGS concordance")

# Result field is 'text' (not 'content') — similarity score 0-1, distance 0-2
```

## Adding Drawers (via MCP)

```python
mcp_mempalace_add_drawer(
    wing="hermes-infra",       # or "ofs-onyx-ma"
    room="openclaw",           # existing or new room
    content="# Title\nCurated knowledge...",
    added_by="rocky"
)
```

**Room creation:** Wings and rooms are created automatically on first use — no pre-creation needed.

## Manual Operations (Python, no Hermes restart needed)

```bash
# Run seeding/query scripts directly in Hermes venv:
MEMPALACE_PALACE_PATH=/home/ubuntu/.mempalace/palace \
MEMPALACE_EMBEDDING_MODEL=minilm \
/home/ubuntu/.hermes/hermes-agent/.venv/bin/python -c "
import sys; sys.path.insert(0, '/home/ubuntu/.hermes/hermes-agent/.venv/lib/python3.11/site-packages')
import mempalace.mcp_server as mcp
result = mcp.tool_search(query='your query', limit=3)
print(result)
"
```

```bash
# CLI search:
MEMPALACE_PALACE_PATH=/home/ubuntu/.mempalace/palace \
/home/ubuntu/.hermes/hermes-agent/.venv/bin/mempalace search "your query"

# Mine documents into palace:
MEMPALACE_PALACE_PATH=/home/ubuntu/.mempalace/palace \
/home/ubuntu/.hermes/hermes-agent/.venv/bin/mempalace mine /path/to/docs --mode extract --wing ofs-onyx-ma
```

## Ingest Documents

```bash
# PDF, DOCX, PPTX, XLSX, RTF, EPUB — installed with mempalace[extract]
mempalace mine /path/to/docs --mode extract --wing ofs-onyx-ma --room abs-docs
```

## Upgrade to Multilingual Embedding (future)

Switch from minilm to embeddinggemma (300MB download, better cross-lingual recall):
1. `"embedding_model": "embeddinggemma"` in `~/.mempalace/config.json`
2. `MEMPALACE_EMBEDDING_MODEL=embeddinggemma mempalace repair rebuild-index`
3. Update MCP env in `~/.hermes/config.yaml`

## Pitfalls

- **Hermes restart required** before `mcp_mempalace_*` tools appear — they're registered at startup
- **Response field is `text`** not `content` when parsing search results
- **Linuxbrew hangs** on EC2 — always use `/home/ubuntu/.hermes/hermes-agent/.venv/bin/python` or `/usr/bin/python3`
- **ChromaDB 1.5.x + embeddinggemma bug** was fixed in v3.4.0 — if search returns empty on fresh install, check version
- **`init` command is interactive** — pipe `echo ""` to accept defaults: `echo "" | mempalace init /path --no-llm`
- **Daedalus has its own separate MemPalace** at `~/.openclaw/workspace-daedalus/mempalace/` — don't confuse the two

## What Belongs in MemPalace vs. Skills vs. Memory

| Type | Goes in |
|------|---------|
| How to do a procedure (steps, commands) | **Skill** |
| Who Paul is, preferences, patterns | **Honcho** |
| Project-specific reference knowledge | **MemPalace** |
| Infrastructure config & gotchas | **MemPalace** |
| Ingested documents (ABS PDFs, API docs) | **MemPalace** |
| Stable facts about tools/environment | **Memory** |
