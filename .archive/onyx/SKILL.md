---
name: onyx
description: Query the OFS Onyx RAG platform (self-hosted Danswer) for indexed Market Assessment reports. Use when asked about OFS MA research, comparable developments, seniors living market data, catchment analysis, or what OFS has previously recommended for similar sites. Covers retirement living (RL), residential aged care (RAC), assisted living (AL), and land lease communities (LLC).
---

# Onyx RAG Skill — OFS Market Assessment Platform

Query the self-hosted Onyx/Danswer RAG platform at onyx.onefellswoop.com.au, which has indexed OFS Market Assessment reports as source documents.

## Credentials

Stored in 1Password — Paul's Vault — "Onyx RAG API — Rocky" (id: nhnsllkxtgrzhmarvkihxvc6pa)

```bash
ONYX_KEY=$(op item get nhnsllkxtgrzhmarvkihxvc6pa --vault "Paul's Vault" --fields credential --reveal 2>/dev/null)
BASE_URL="https://onyx.onefellswoop.com.au/api"
```

## Personas

| ID | Name | Document Set | Best for |
|----|------|-------------|---------|
| 1 | R&A MA Assistant | Market Assessment document set | Research questions, comparable sites, market data |
| 3 | Market Assessment Writer | Market sample reports | Writing MA sections, report style/format |

Default to **Persona 1** for research queries. Use **Persona 3** for drafting report text.

## Query Pattern

Three steps: create session → send message → parse SSE stream.

```bash
# 1. Create chat session
SESSION=$(curl -s -X POST "$BASE_URL/chat/create-chat-session" \
  -H "Authorization: Bearer $ONYX_KEY" \
  -H "Content-Type: application/json" \
  -d '{"persona_id": 1}' \
  | python3 -c "import json,sys; print(json.load(sys.stdin).get('chat_session_id',''))")

# 2. Send message and collect response
RESPONSE=$(curl -s -X POST "$BASE_URL/chat/send-chat-message" \
  -H "Authorization: Bearer $ONYX_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"chat_session_id\": \"$SESSION\", \"message\": \"Your question here\", \"parent_message_id\": null, \"prompt_id\": null}" \
  | python3 -c "
import sys, json
full = ''
docs = []
for line in sys.stdin:
    line = line.strip()
    if not line: continue
    try:
        d = json.loads(line)
        obj = d.get('obj', {})
        t = obj.get('type','')
        if t == 'message_delta':
            full += obj.get('content','')
        elif t == 'message_start':
            docs = obj.get('final_documents') or []
    except: pass
print('ANSWER:', full)
print()
print('SOURCES:')
for doc in docs[:10]:
    print(' -', doc.get('semantic_identifier','?'))
")
echo "$RESPONSE"
```

## Indexed Documents

50+ OFS Market Assessment PDFs covering:
- Retirement Living (RL), Residential Aged Care (RAC), Assisted Living (AL), Land Lease Communities (LLC)
- States: VIC, NSW, QLD, SA
- Example docs: LINE7828 Geelong, TULI7947 Oran Park, CAPA7833 Sacred Heart Kew, GOOC7984 Blackburn South

## Good Query Types

- "What have OFS MAs recommended for retirement villages in [suburb/region]?"
- "What is the typical ILU price range for [area] based on OFS reports?"
- "Find comparable seniors living developments in [suburb]"
- "What does OFS research say about demand drivers for [type] in [state]?"
- "Summarise key findings from the [project code] market assessment"

## Multi-turn Conversations

For follow-up questions, pass `parent_message_id` from the previous response:

```bash
# Get the assistant message ID from the first response
PARENT_ID=$(curl -s -X POST "$BASE_URL/chat/send-chat-message" ... | python3 -c "
import sys, json
for line in sys.stdin:
    try:
        d = json.loads(line.strip())
        if d.get('obj',{}).get('type') == 'message_start':
            print(d.get('reserved_assistant_message_id',''))
    except: pass
")
# Then use parent_message_id: $PARENT_ID in next request
```

## Notes

- Response streaming uses NDJSON (one JSON object per line)
- `obj.type` values: `reasoning_start`, `reasoning_delta`, `reasoning_done`, `message_start`, `message_delta`
- Source documents appear in `obj.final_documents` on the `message_start` event
- Rate limits: Vertex AI backend — if 429 error, wait a few seconds and retry
- Persona 3 (Market Assessment Writer) has a strict "no filler" system prompt — responses are formal report style
