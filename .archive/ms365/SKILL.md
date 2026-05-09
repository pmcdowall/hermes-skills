---
name: ms365
description: Access Microsoft 365 for One Fell Swoop via Microsoft Graph API. Use when asked to read or send email, check or create calendar events, access SharePoint, or read Teams messages. Pre-configured for rocky@onefellswoop.com.au. Covers: email (inbox, send, reply, search), calendar (events, create, update), Teams (channels, messages), SharePoint (sites, files).
---

# Microsoft 365 Skill — OFS Studio Agent

Access M365 via Microsoft Graph API using app-only (client credentials) auth.
Tokens expire after 3600s — always fetch a fresh token at the start of each M365 operation.

## Credentials

Stored in 1Password — Paul's Vault — "MS365 Azure App — Rocky" (id: 7omxmms2z22ac6fd4hdbk5bveq)

```
Tenant ID:  8a80ec0f-2e54-4fc7-9c59-1537d57d22ce
Client ID:  01cb25d3-be0f-4130-a049-9cb7327d17b4
User:       rocky@onefellswoop.com.au
Graph base: https://graph.microsoft.com/v1.0/users/rocky@onefellswoop.com.au
```

## Fetch Token

```bash
MS365_SECRET=$(op item get 7omxmms2z22ac6fd4hdbk5bveq --vault "Paul's Vault" --fields credential --reveal 2>/dev/null)
TOKEN=$(curl -s -X POST \
  "https://login.microsoftonline.com/8a80ec0f-2e54-4fc7-9c59-1537d57d22ce/oauth2/v2.0/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials&client_id=01cb25d3-be0f-4130-a049-9cb7327d17b4&client_secret=${MS365_SECRET}&scope=https%3A%2F%2Fgraph.microsoft.com%2F.default" \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['access_token'])")
```

## Email Operations

### Read inbox
```bash
curl -s "https://graph.microsoft.com/v1.0/users/rocky@onefellswoop.com.au/mailFolders/inbox/messages?\$top=20&\$orderby=receivedDateTime desc&\$select=subject,from,receivedDateTime,isRead,bodyPreview" \
  -H "Authorization: Bearer $TOKEN"
```

### Search email
```bash
curl -s "https://graph.microsoft.com/v1.0/users/rocky@onefellswoop.com.au/messages?\$search=\"{query}\"&\$select=subject,from,receivedDateTime,bodyPreview" \
  -H "Authorization: Bearer $TOKEN"
```

### Read a specific message (full body)
```bash
curl -s "https://graph.microsoft.com/v1.0/users/rocky@onefellswoop.com.au/messages/{messageId}" \
  -H "Authorization: Bearer $TOKEN"
```

### Send email
```bash
curl -s -X POST "https://graph.microsoft.com/v1.0/users/rocky@onefellswoop.com.au/sendMail" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "message": {
      "subject": "Subject here",
      "body": { "contentType": "HTML", "content": "<p>Body here</p>" },
      "toRecipients": [{ "emailAddress": { "address": "recipient@example.com" } }],
      "ccRecipients": []
    },
    "saveToSentItems": true
  }'
```

### Reply to a message
```bash
curl -s -X POST "https://graph.microsoft.com/v1.0/users/rocky@onefellswoop.com.au/messages/{messageId}/reply" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "message": {}, "comment": "Reply text here" }'
```

## Calendar Operations

### List upcoming events
```bash
curl -s "https://graph.microsoft.com/v1.0/users/rocky@onefellswoop.com.au/calendarView?startDateTime={ISO8601}&endDateTime={ISO8601}&\$select=subject,start,end,organizer,location" \
  -H "Authorization: Bearer $TOKEN"
```

### Create event
```bash
curl -s -X POST "https://graph.microsoft.com/v1.0/users/rocky@onefellswoop.com.au/events" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "subject": "Event title",
    "start": { "dateTime": "2026-04-03T10:00:00", "timeZone": "Australia/Melbourne" },
    "end":   { "dateTime": "2026-04-03T11:00:00", "timeZone": "Australia/Melbourne" },
    "attendees": [{ "emailAddress": { "address": "paul@onefellswoop.com.au" }, "type": "required" }]
  }'
```

## Teams Operations

**Important:** App-only (client credentials) can READ Teams messages but CANNOT SEND to channels or DMs.
Sending requires delegated permissions or an incoming webhook. Use email for outbound comms.

### Rocky's IDs
- Rocky user ID: `e519a76f-1753-4576-8282-5ea51b94c9a2`
- Rocky ↔ Paul DM chat ID: `19:43c180ee-0b4b-476e-b625-6b710850aa82_e519a76f-1753-4576-8282-5ea51b94c9a2@unq.gbl.spaces`
- Paul user ID: `43c180ee-0b4b-476e-b625-6b710850aa82`
- OFS General team ID: `273470f2-03e2-409e-921a-e720a7418e8e`

### List Rocky's chats
```bash
curl -s "https://graph.microsoft.com/v1.0/users/e519a76f-1753-4576-8282-5ea51b94c9a2/chats?\$expand=members" \
  -H "Authorization: Bearer $TOKEN"
```

### Read DM messages (Rocky ↔ Paul)
```bash
CHAT_ID="19:43c180ee-0b4b-476e-b625-6b710850aa82_e519a76f-1753-4576-8282-5ea51b94c9a2@unq.gbl.spaces"
curl -s "https://graph.microsoft.com/v1.0/chats/$CHAT_ID/messages" \
  -H "Authorization: Bearer $TOKEN"
```

### List joined teams
```bash
curl -s "https://graph.microsoft.com/v1.0/users/rocky@onefellswoop.com.au/joinedTeams" \
  -H "Authorization: Bearer $TOKEN"
```

### List channels in a team
```bash
curl -s "https://graph.microsoft.com/v1.0/teams/{teamId}/channels" \
  -H "Authorization: Bearer $TOKEN"
```

### Read channel messages
```bash
curl -s "https://graph.microsoft.com/v1.0/teams/{teamId}/channels/{channelId}/messages" \
  -H "Authorization: Bearer $TOKEN"
```

## SharePoint Operations

### Search for a SharePoint site
```bash
curl -s "https://graph.microsoft.com/v1.0/sites?search={query}" \
  -H "Authorization: Bearer $TOKEN"
```

### List document library contents
```bash
curl -s "https://graph.microsoft.com/v1.0/sites/{siteId}/drives" \
  -H "Authorization: Bearer $TOKEN"
curl -s "https://graph.microsoft.com/v1.0/sites/{siteId}/drives/{driveId}/root/children" \
  -H "Authorization: Bearer $TOKEN"
```

## Notes

- Always use `saveToSentItems: true` when sending email so there's an audit trail
- Email body: prefer `HTML` contentType for formatted output, `Text` for simple messages
- Timezone for calendar: always specify `Australia/Melbourne`
- When composing follow-up emails as Studio Agent, sign off as "Rocky | OFS Studio Agent"
- Paul's email: paul@onefellswoop.com.au
