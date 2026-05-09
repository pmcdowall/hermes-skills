---
name: ofs-platforms
description: "OFS business platform integrations: WIPlist (jobs/estimates/invoices), Trello (studio workflow boards), Google Drive/Sheets (hosting records), Microsoft 365 (email/calendar/Teams via Graph API), Onyx RAG (market assessment research). All pre-configured for One Fell Swoop (paul@onefellswoop.com.au). Load when asked about jobs, invoices, boards, email, calendar, or OFS research."
version: 1.0.0
author: Hermes Agent
metadata:
  hermes:
    tags: [OFS, WIPlist, Trello, Google Drive, Microsoft 365, Onyx, business-platforms, one-fell-swoop]
    related_skills: [salesforce, openclaw, shared-agent-context]
---

# OFS Business Platform Integrations

Unified index for One Fell Swoop's business toolset. All credentials stored in 1Password — Paul's Vault.

| Platform | Use for | Credential ID | Full reference |
|----------|---------|---------------|----------------|
| **WIPlist** | Jobs, estimates, invoices, time entries, purchases, tasks, team schedule | `x4ldpodazg42ilwwczuswu5ce4` | [references/wiplist.md](references/wiplist.md) |
| **Trello** | Studio workflow boards, card status, moving cards, project summaries | `nfwjhq446tnme7o2qqrk4bevni` | [references/trello.md](references/trello.md) |
| **Google Drive/Sheets** | Hosting record spreadsheets, Drive file search | Service account `~/.openclaw/google-service-account.json` | [references/google-drive.md](references/google-drive.md) |
| **Microsoft 365** | Email (inbox/send/reply/search), calendar events, Teams messages, SharePoint | `7omxmms2z22ac6fd4hdbk5bveq` | [references/ms365.md](references/ms365.md) |
| **Onyx RAG** | OFS Market Assessment research, comparable developments, seniors living data | `nhnsllkxtgrzhmarvkihxvc6pa` | [references/onyx.md](references/onyx.md) |

> **Salesforce** (contacts, leads, opportunities, Apex/Flows) → see the `salesforce` skill (too large for inline reference).

---

## WIPlist — Quick Reference

Base URL: `https://app.wiplist.io` | Solution ID: `4revg6` | User: `paul@onefellswoop.com.au`

```bash
WIPLIST_PASS=$(op item get x4ldpodazg42ilwwczuswu5ce4 --vault "Paul's Vault" --fields credential --reveal 2>/dev/null)
TOKEN=$(curl -s -X POST "https://app.wiplist.io/api/v1/login" \
  -H "Content-Type: application/json" -H "Accept: application/json" -H "X-Solution: 4revg6" \
  -d "{\"email\":\"paul@onefellswoop.com.au\",\"password\":\"$WIPLIST_PASS\"}" \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['data']['token'])")
# Always include on every request: Authorization: Bearer $TOKEN, X-Solution: 4revg6, Accept: application/json
```

Key endpoints: `/api/v1/jobs`, `/api/v1/estimates`, `/api/v1/invoices`, `/api/v1/times`, `/api/v1/tasks`, `/api/v1/companies`, `/api/v1/users`  
Job status codes: `200`=active, `210`=in progress, `215`=review, `220`=delivered, `300`=archived  
Full reference with all endpoint examples → [`references/wiplist.md`](references/wiplist.md)

---

## Trello — Quick Reference

API key: `791f899c1b00dea6f4b66f7f33598026` | Base URL: `https://api.trello.com/1`

```bash
TRELLO_KEY="791f899c1b00dea6f4b66f7f33598026"
TRELLO_TOKEN=$(op item get nfwjhq446tnme7o2qqrk4bevni --vault "Paul's Vault" --fields Token --reveal 2>/dev/null)
# Append ?key=$TRELLO_KEY&token=$TRELLO_TOKEN to every request
```

Key boards: Studio workflow `690a62fbe5a80149bc54eedc` | Studio allocations `691ff75e6c3bb2055059cf55`  
Full reference with list IDs and card operations → [`references/trello.md`](references/trello.md)

---

## Google Drive/Sheets — Quick Reference

Service account: `rocky-agent@rocky-agent-493502.iam.gserviceaccount.com`  
Key file: `~/.openclaw/google-service-account.json`

```python
from google.oauth2 import service_account
from googleapiclient.discovery import build
SCOPES = ['https://www.googleapis.com/auth/drive', 'https://www.googleapis.com/auth/spreadsheets']
creds = service_account.Credentials.from_service_account_file('/home/ubuntu/.openclaw/google-service-account.json', scopes=SCOPES)
drive = build('drive', 'v3', credentials=creds)
sheets = build('sheets', 'v4', credentials=creds)
```

Full reference with sheet IDs and common operations → [`references/google-drive.md`](references/google-drive.md)

---

## Microsoft 365 — Quick Reference

Tenant: `8a80ec0f-2e54-4fc7-9c59-1537d57d22ce` | Client: `01cb25d3-be0f-4130-a049-9cb7327d17b4`  
User: `rocky@onefellswoop.com.au` | Graph base: `https://graph.microsoft.com/v1.0/users/rocky@onefellswoop.com.au`

```bash
MS365_SECRET=$(op item get 7omxmms2z22ac6fd4hdbk5bveq --vault "Paul's Vault" --fields credential --reveal 2>/dev/null)
TOKEN=$(curl -s -X POST "https://login.microsoftonline.com/8a80ec0f-2e54-4fc7-9c59-1537d57d22ce/oauth2/v2.0/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials&client_id=01cb25d3-be0f-4130-a049-9cb7327d17b4&client_secret=${MS365_SECRET}&scope=https%3A%2F%2Fgraph.microsoft.com%2F.default" \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['access_token'])")
```

⚠️ Teams: app-only auth can READ but NOT SEND to channels/DMs — use email for outbound.  
Full reference with email/calendar/Teams/SharePoint operations → [`references/ms365.md`](references/ms365.md)

---

## Onyx RAG — Quick Reference

Base URL: `https://onyx.onefellswoop.com.au/api`  
Personas: `1` = R&A MA Assistant (research), `3` = Market Assessment Writer (report drafting)

```bash
ONYX_KEY=$(op item get nhnsllkxtgrzhmarvkihxvc6pa --vault "Paul's Vault" --fields credential --reveal 2>/dev/null)
# Three steps: create session → send message → parse SSE stream
SESSION=$(curl -s -X POST "https://onyx.onefellswoop.com.au/api/chat/create-chat-session" \
  -H "Authorization: Bearer $ONYX_KEY" -H "Content-Type: application/json" \
  -d '{"persona_id": 1}' | python3 -c "import json,sys; print(json.load(sys.stdin).get('chat_session_id',''))")
```

Good queries: comparable developments, ILU price ranges, demand drivers by type/state, MA summaries by project code.  
Full reference with SSE parsing and indexed document list → [`references/onyx.md`](references/onyx.md)
